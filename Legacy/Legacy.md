
# Walkthrough: 

This is my first ever Windows machine... Let's see how it goes.

### Nmap Scans: 

```
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```


```
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: mean: 5d00h27m38s, deviation: 2h07m16s, median: 4d22h57m38s
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: a2:de:ad:af:9d:e1 (unknown)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2026-07-11T12:21:26+03:00
```

We have a Windows XP machine running. Open Ports are the RPC on 135 and SMB on 139/445. 

Experience and suspicion (the dated Windows host has open SMB ports) tells me that the host is likely vulnerable to EternalBlue. We run a scanner from the msfconsole `auxiliary(scanner/smb/smb_ms17_010)`:

![image](legacy_bluecheck.png)

The host is vulnerable! Let's move on to using the EternalBlue exploit against the target:

![image](Legacy_etrnlblew.png)

`NT Authority\SYSTEM` generally has permissions to access any user's profile. It's almost the equivalent of root. 

The Windows XP OS keeps its user directories in `Document and Settings`. This is the Windows XP equivalent of the `/home` directory on linux. 

![image](legacy_user.txt.png)

Now for the root flag, which on Windows we will assume as Administrator. This is going to be easy as the EternalBlue exploit already did the hard work for us:

![image](legacy_rootflag.png)

---

# Alternative Route 

`sudo nmap --script smb-vuln* -p 139,445 10.129.227.181`

```
PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_smb-vuln-ms10-054: false
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
```

The Nmap scan tells us that the host is vulnerable to:

**ms08-067** 
**CVE-2008-4250**

This is a critical, "wormable" Remote Code Execution (RCE) vulnerability in the Microsoft Windows Server service. Unauthenticated attackers could send specially crafted `RPC` requests to take full system control. It famously powered devastating malware like the Conficker worm and Stuxnet.

Again, this is where Metasploit comes in handy for exploitation:

![image](legacy_ms08-067.png)

# Summary:

The target is a Windows XP machine (“LEGACY”) with SMB (ports 139/445) and MSRPC (135) exposed, showing typical legacy Windows configuration with SMBv1, no SMB signing, and poor security defaults. Nmap identifies it as vulnerable to both MS08-067 (RPC Server Service RCE) and MS17-010 (EternalBlue), meaning it can be exploited remotely without authentication to achieve SYSTEM-level access. Exploitation would grant full control of the host, allowing access to all user directories (stored under `C:\Documents and Settings` on XP) and any sensitive files, effectively compromising the entire system.