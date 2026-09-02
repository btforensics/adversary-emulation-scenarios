# Quantum Ransomware Scenario: README

**Quick Facts**
- **Time:** 2–6 hours (depends on scope and lateral movement depth)
- **Tools:** Meterpreter, Cobalt Strike, ISO mount, Mimikatz, ADFind, rclone
- **Scope:** Full ATT&CK chain from spear phishing → ransomware deployment
- **Lab:** 5+ VMs (1 attacker, 4+ targets, 1 file server)

---

## What Is This?

A **hands-on training scenario** that simulates a realistic Quantum ransomware intrusion based on DFIR report analysis:
- Initial access via spear phishing (ISO image + IceID malware)
- Meterpreter beacon establishment & persistence
- Network reconnaissance & credential extraction
- Lateral movement across domain
- Data exfiltration via rclone
- Ransomware deployment & encryption

Uses **generic red-team tooling + IceID emulation** to teach tactics/techniques, not a replica of actual Quantum malware.

---

## Lab Setup

| Host | OS | Role | IP | Notes |
|------|----|----|-----|--------|
| **MORPHEUS** | Kali Linux | Attacker (C2) | 172.20.43.5 | Meterpreter/Cobalt Strike listener |
| **TRINITY** | Windows 11 | Initial target | 172.20.43.X | Initial compromise via phishing |
| **NEO** | Windows 11 | Secondary target | 172.20.43.X | Lateral movement target |
| **NEB** | Windows Server | File server | 172.20.43.X | Data staging & exfiltration |
| **ZION** | Windows Server | Domain controller | 172.20.43.X | AD enumeration & credential dumping |

**Domain:** `<DOMAIN_NAME>` | **Admin:** `Administrator` / `<LAB_ADMIN_PASSWORD>`

**Pre-cached Tools:** `/home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/`

---

## Attack Flow (8 Phases)

```
PHASE 1: Reconnaissance (T1592, T1589)
└─ Information gathering on targets

PHASE 2: Weaponization (T1566, T1588)
└─ Create malicious ISO with IceID DLL + shortcut LNK

PHASE 3: Delivery (T1598, T1193)
└─ Spear phishing email with ISO attachment

PHASE 4: Exploitation (T1204, T1566.001)
└─ User opens ISO, executes shortcut, loads IceID DLL

PHASE 5: Installation (T1547, T1053)
└─ Meterpreter beacon + scheduled task persistence

PHASE 6: Command & Control (T1087, T1018, T1083, T1003)
└─ Domain enumeration, host discovery, credential extraction

PHASE 7: Exfiltration (T1560, T1567)
└─ Archive sensitive data, upload via rclone to cloud

PHASE 8: Impact (T1486, T1565)
└─ Deploy ransomware via psexec, encrypt all files
```

---

## ATT&CK Techniques Demonstrated

| Phase | Technique ID | What It Does |
|-------|-------------|-------------|
| Reconnaissance | T1592.004, T1589.001 | Gather email/system info on targets |
| Weaponization | T1566.001, T1588.003 | Craft malicious ISO + DLL |
| Delivery | T1598.001, T1193 | Spear phishing email |
| Exploitation | T1204.002, T1566.001 | User executes LNK file |
| Installation | T1547.001, T1053.005 | Scheduled task persistence |
| Discovery | T1087.002, T1018, T1083, T1135 | Domain/host/share enumeration via PowerShell |
| Credential Access | T1003.001, T1110 | Mimikatz, logon password dump |
| Lateral Movement | T1021.001, T1047 | psexec, WMI execution |
| Collection | T1560.001, T1123 | Archive & collect data |
| Exfiltration | T1567, T1048 | Upload via rclone/C2 |
| Impact | T1486 | Ransomware encryption |

---

## Tools & Downloads

Before starting, download and prepare these tools on MORPHEUS:

| Tool | Purpose | Download Link |
|------|---------|---------------|
| **ImgBurn** | ISO image creation | https://www.imgburn.com/ |
| **Proton Mail** | Phishing email delivery | https://account.proton.me |
| **Metasploit Framework** | Meterpreter & exploitation | `apt install metasploit-framework` (Kali default) |
| **Mimikatz** | Credential dumping | [GitHub Release](https://github.com/gentilkiwi/mimikatz/releases) |
| **ADFind** | Active Directory enumeration | [JoeWare ADFind](http://www.joeware.net/freetools/tools/adfind/) |
| **psexec** | Lateral movement (SMB) | [Sysinternals psexec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) |
| **rclone** | Data exfiltration to cloud | https://rclone.org/downloads/ |
| **evil-winrm** | WinRM lateral movement | `gem install evil-winrm` |
| **Cobalt Strike** (optional) | Advanced C2 alternative | [Cobalt Strike](https://www.cobaltstrike.com/) |

**Pre-cached location:** `/home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/`

---

## Quick Prerequisites

Before you start:

- [ ] 5+ VMs (lab network isolated from production)
- [ ] MORPHEUS: Kali Linux with msfconsole, Cobalt Strike optional
- [ ] TRINITY/NEO/NEB/ZION: Windows, domain-joined to `<DOMAIN_NAME>`
- [ ] All hosts synchronized time
- [ ] Vision One agent monitoring (or equivalent EDR)
- [ ] Pre-cached tools downloaded to `/home/test/Desktop/Adversary_Profiles/Quantum_Ransomware/`
- [ ] Test data staged on TRINITY/NEB (sensitive documents)
- [ ] Temporary Proton Mail account created at https://account.proton.me
- [ ] MEGA account for rclone exfiltration (https://mega.io or similar)

**Key check:** Verify MORPHEUS can reach all targets on port 135/139/445 (SMB), 3389 (RDP), 5985 (WinRM) before PHASE 5.

---

## Important Notes

⚠️ **All sample data is fictional:** Employee names, emails, company data in PHASE 6–7 are lab-generated, not real.

⚠️ **Credentials are placeholders:** Replace `<LAB_ADMIN_PASSWORD>`, `<PROTON_EMAIL>`, `<PROTON_PASSWORD>` with your actual lab credentials — never reuse real passwords.

⚠️ **This is a training emulation:** Uses generic tools (Meterpreter, Mimikatz, psexec) to emulate Quantum TTPs; not a fidelity replica of actual Quantum malware/C2.

⚠️ **Email account requirement:** You will need a temporary Proton Mail account (or equivalent) for phishing delivery. Do not use production email accounts.

⚠️ **Ransomware deployment is destructive:** PHASE 8 will encrypt files. Run on isolated lab only.

---

## Next Steps

1. **Read [`Quantum_Ransomware_Execution_Plan.md`](./Quantum_Ransomware_Execution_Plan.md)** — start at Infrastructure Setup
2. **Create test data** — stage documents on TRINITY/NEB (PHASE 1 setup)
3. **Execute attack chain** — run PHASE 2–5 (weaponization → persistence, ~1–2 hours)
4. **Discovery & lateral movement** (optional) — run PHASE 6–7 for credential dumping & exfiltration (1–2 hours)
5. **Ransomware deployment** (optional/final) — run PHASE 8 for full impact demonstration

---

## References & Resources

**Threat Intelligence & ATT&CK**
- [DFIR Report: Quantum Ransomware](https://thedfirreport.com/)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Quantum Ransomware Profile](https://attack.mitre.org/software/)

**Tools & Documentation**
- [Metasploit Framework](https://docs.metasploit.com/)
- [Meterpreter Handler](https://docs.metasploit.com/docs/using-metasploit/basics/handlers.html)
- [Meterpreter & Cobalt Strike Documentation](https://www.cobaltstrike.com/)
- [Mimikatz GitHub Repository](https://github.com/gentilkiwi/mimikatz)
- [ADFind by JoeWare](http://www.joeware.net/freetools/tools/adfind/)
- [Sysinternals psexec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec)
- [rclone Documentation](https://rclone.org/)
- [evil-winrm GitHub](https://github.com/Hackplayers/evil-winrm)

**Email & Exfiltration**
- [Proton Mail](https://account.proton.me)
- [MEGA Cloud Storage](https://mega.io/)
- [rclone MEGA Backend](https://rclone.org/mega/)

**ISO & Media Creation**
- [ImgBurn](https://www.imgburn.com/)
- [Linux mkisofs](https://linux.die.net/man/8/mkisofs)

**Educational Resources**
- [CTID Adversary Emulation Library](https://github.com/center-for-threat-informed-defense/adversary_emulation_library)
- [Trend Micro Vision One XDR Documentation](https://www.trendmicro.com/en_us/business/products/detection-response/xdr.html)

---

**For Training Use Only — Authorized Personnel Only**
