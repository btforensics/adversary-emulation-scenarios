# Nefilim Ransomware Adversary Emulation

**Source:** <cite index="3-1">This guide is based on the Nefilim Ransomware Adversary Emulation Guide from the [GPENBuddyT Wiki](https://github.com/btforensics/GPENBuddyT/wiki/Nefilim-Ransomware-Adversary-Emulation-Guide), created by Buddy Tancio in reference to the GPEN GOLD research paper "Adversary Emulation: Nefilim Ransomware vs. Security Onion"</cite>

**Quick Facts**
- **Time:** 4–8 hours
- **Tools:** Metasploit, MSFvenom, Mimikatz, PowerSploit, Impacket, BloodHound, 7zip, MEGASync
- **Scope:** Citrix RCE → persistence → privilege escalation → credential theft → lateral movement → data exfiltration → ransomware encryption
- **Lab:** 5 VMs (1 attacker, 4 targets)

---

## What Is This?

A **comprehensive ransomware incident simulation** based on Nefilim (also known as Nemty) attack chain:
- Initial access via Citrix Application Delivery Controller (ADC) RCE
- DLL injection for persistent C2 backdoor
- Privilege escalation via kernel exploits
- Defense evasion (AV bypass, pass-the-hash)
- Comprehensive credential dumping and AD enumeration
- Multi-stage lateral movement with tool transfer
- Data collection and cloud exfiltration
- Volume shadow copy deletion and full-disk encryption

Focuses on the **complete kill chain from exploitation through financial impact**, with realistic offensive operational security.

---

## Lab Setup

| Host | OS | Role | IP |
|------|----|----|-----|
| **MORPHEUS** | Kali Linux | Attacker (C2) | 192.168.150.50 |
| **NEO** | Windows 11 | Initial target | 192.168.150.21 |
| **TRINITY** | Windows 11 | Secondary target | 192.168.150.22 |
| **NEB** | Server 2025 | File server | 192.168.150.20 |
| **ZION** | Server 2025 | Domain controller | 192.168.150.10 |

**Domain:** `corp.lab` | **Admin:** `Administrator` / `<LAB_ADMIN_PASSWORD>` | **SSH:** `root` / `<LAB_SSH_PASSWORD>`

---

## Attack Flow (11 Stages)

```
STAGE 1: Initial Access (T1190)
└─ Citrix ADC RCE via CVE-2019-19781

STAGE 2: Persistence (T1574.002)
└─ DLL side-loading with Meterpreter reverse_tcp

STAGE 3: Privilege Escalation (T1068)
└─ Kernel exploit (CVE-2017-0213)

STAGE 4: Defense Evasion (T1550, T1562.001, T1070.004)
└─ Pass-the-hash, AV disable, file deletion

STAGE 5: Credential Access (T1003, T1555)
└─ Mimikatz dumping, password store extraction

STAGE 6: Discovery (T1082, T1046, T1018, T1482, T1083, T1049)
└─ System fingerprinting, network scanning, AD enumeration, file discovery

STAGE 7: Lateral Movement (T1570, T1021.002)
└─ Batch file transfer, remote execution via PsExec

STAGE 8: Collection (T1560.001)
└─ Archive sensitive data with 7zip

STAGE 9: Command & Control (T1071)
└─ Cobalt Strike beacon (simulated via APTSimulator)

STAGE 10: Exfiltration (T1567.002)
└─ Cloud storage via MEGASync

STAGE 11: Impact (T1490, T1486)
└─ Shadow copy deletion, AES encryption deployment
```

---

## Key Differences from FIN6

- **Attack vector:** Citrix ADC public-facing app RCE (vs. phishing macro)
- **Persistence:** DLL injection with Meterpreter (vs. scheduled tasks)
- **Credential theft:** Offline NTLM dumping + browser password extraction
- **Scope:** Full domain enumeration + multi-server encryption
- **Exfiltration:** Cloud storage integration (MEGASync)
- **Impact:** Complete file encryption across network shares

---

## Files in This Scenario

| File | What It Has |
|------|------------|
| **Nefilim_README.md** | This overview (you are here) |
| **Nefilim_Execution_Plan.md** | Step-by-step execution (infrastructure → encryption) |

---

## ATT&CK Techniques Demonstrated

| Phase | Technique ID | Technique Name |
|-------|-------------|-------------|
| Initial Access | T1190 | Exploit Public-Facing Application |
| Persistence | T1574.002 | Hijack Execution Flow: DLL Side-Loading |
| Privilege Escalation | T1068 | Exploitation for Privilege Escalation |
| Defense Evasion | T1550 | Use Alternate Authentication Material: Pass the Hash |
| Defense Evasion | T1562.001 | Impair Defenses: Disable or Modify Tools |
| Defense Evasion | T1070.004 | Indicator Removal on Host: File Deletion |
| Credential Access | T1003 | OS Credential Dumping |
| Credential Access | T1555 | Credentials from Password Stores |
| Discovery | T1082 | System Information Discovery |
| Discovery | T1046 | Network Service Scanning |
| Discovery | T1018 | Remote System Discovery |
| Discovery | T1482 | Domain Trust Discovery |
| Discovery | T1083 | File and Directory Discovery |
| Discovery | T1049 | System Network Connections Discovery |
| Lateral Movement | T1570 | Lateral Tool Transfer |
| Lateral Movement | T1021.002 | Remote Services: SMB/Windows Admin Shares |
| Collection | T1560.001 | Archive Collected Data: Archive via Utility |
| Command & Control | T1071 | Application Layer Protocol |
| Exfiltration | T1567.002 | Exfiltration Over Web Service: Exfiltration to Cloud Storage |
| Impact | T1490 | Inhibit System Recovery |
| Impact | T1486 | Data Encrypted for Impact |

---

## Quick Prerequisites

Before you start:

- [ ] 5 VMs (lab network isolated, NOT connected to corporate network)
- [ ] MORPHEUS: Kali with full Metasploit framework installation
- [ ] NEO/TRINITY/NEB/ZION: domain-joined to corp.lab
- [ ] All hosts synchronized time (critical for Kerberos)
- [ ] Test files created in user documents and shared folders (STEP 0 in execution plan)
- [ ] Sensitive file server shares set up (NEB)
- [ ] Lab snapshots taken before Stage 11 (encryption is destructive)

**WARNING:** Ransomware binary will encrypt files. Do this in an **isolated lab environment only**. **VM snapshots are mandatory before executing Stage 11.**

---

## Important Notes

⚠️ **Destructive:** This scenario **actually encrypts files** using a ransomware emulator. Do NOT run in production or non-isolated lab.

⚠️ **Lab credentials are placeholders:** Replace `<LAB_ADMIN_PASSWORD>` and `<LAB_SSH_PASSWORD>` with your actual lab values.

⚠️ **Antivirus/EDR evasion:** This scenario includes techniques to bypass Windows Defender and endpoint detection — part of the training value, but adds complexity. Ensure your lab VMs have Defender enabled for realistic telemetry.

⚠️ **Time investment:** Full chain (Citrix RCE → encryption) takes 4–8 hours depending on network speed and tool execution time.

⚠️ **Network isolation critical:** Use a dedicated lab VLAN with no internet access except where explicitly allowed (MEGASync exfiltration path should still be isolated).

---

## Next Steps

1. **Read [`Nefilim_Execution_Plan.md`](./Nefilim_Execution_Plan.md)** — start at Infrastructure Setup
2. **Create test data** — run STEP 0 (populate documents + shares on NEO/TRINITY/NEB)
3. **Stages 1–6 (Enabling):** Initial access → discovery, ~3 hours
4. **Stages 7–9 (Lateral Movement & Exfil):** Lateral movement → C2, ~2 hours
5. **Stage 10–11 (Impact):** Data encryption, ~30 min (DESTRUCTIVE — take snapshots first)

---

## References

- [GPENBuddyT Nefilim Adversary Emulation Guide](https://github.com/btforensics/GPENBuddyT/wiki/Nefilim-Ransomware-Adversary-Emulation-Guide)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Nefilim Ransomware Group Profile](https://attack.mitre.org/groups/G0037/)
- [CVE-2019-19781 Citrix ADC RCE](https://nvd.nist.gov/vuln/detail/CVE-2019-19781)
- [CVE-2017-0213 Windows Kernel Exploit](https://nvd.nist.gov/vuln/detail/CVE-2017-0213)

---

**For Training Use Only — Authorized Personnel Only**
