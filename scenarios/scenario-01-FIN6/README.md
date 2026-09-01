# FIN6 Scenario 1: Ransomware Deployment

**Quick Facts**
- **Time:** 2–5 hours
- **Tools:** Meterpreter, Impacket, Cobalt Strike (simulated), PowerShell
- **Scope:** Credential access → lateral movement → ransomware encryption
- **Lab:** 5 VMs (1 attacker, 4 targets)

---

## What Is This?

A **hands-on ransomware incident simulation** based on FIN6 tactics:
- Initial access via macro or assume-breach
- Discovery + credential extraction
- Lateral movement across domain
- Persistence setup
- Ransomware deployment and encryption

Focuses on the **kill chain leading up to encryption**, including all defensive obstacles (EDR evasion, credential cracking, privilege escalation).

---

## Lab Setup

| Host | OS | Role | IP |
|------|----|----|-----|
| **MORPHEUS** | Kali Linux | Attacker (C2) | 192.168.150.50 |
| **NEO** | Windows 11 | Initial target | 192.168.150.21 |
| **TRINITY** | Windows 11 | Secondary target | 192.168.150.22 |
| **NEB** | Server 2025 | Member server | 192.168.150.20 |
| **ZION** | Server 2025 | Domain controller | 192.168.150.10 |

**Domain:** `corp.lab` | **Admin:** `Administrator` / `<LAB_ADMIN_PASSWORD>` | **SSH:** `root` / `<LAB_SSH_PASSWORD>`

---

## Attack Flow (5 Phases)

```
PHASE 1: Initial Access (T1566 or T1566.001)
└─ Macro-enabled Word doc OR assume-breach on workstation

PHASE 2: Discovery & Enumeration (T1087, T1018, T1083)
└─ Domain/network discovery from initial host

PHASE 3: Credential Access & Cracking (T1003, T1110)
└─ Extract hashes, crack with hashcat on MORPHEUS

PHASE 4: Lateral Movement & Persistence (T1021, T1053)
└─ psexec/wmiexec to secondary hosts, install backdoors

PHASE 5: Encryption & Exfiltration (T1490, T1020)
└─ Deploy ransomware binary, encrypt file shares
```

---

## Key Differences from APT29

- **Objective:** Data destruction (ransomware), not just theft
- **Defense evasion:** Windows Defender bypass, process injection
- **Credential cracking:** Offline NTLM hash cracking (hashcat)
- **File server targeting:** Focus on high-value shares (NEB)
- **Encryption scope:** Local + network drives

---

## Files in This Scenario

| File | What It Has |
|------|------------|
| **README.md** | This overview (you are here) |
| **FIN6_Execution_Plan.md** | Step-by-step execution (infrastructure → encryption) |

---

## ATT&CK Techniques Demonstrated

| Phase | Technique ID | What It Does |
|-------|-------------|-------------|
| Initial Access | T1566.001 | Phishing + macro execution |
| Discovery | T1087.002, T1018, T1083 | Domain enumeration |
| Credential Access | T1003.001, T1110.004 | Hash dumping, offline cracking |
| Persistence | T1547.001, T1053.005 | Registry run key, scheduled task |
| Lateral Movement | T1021.002, T1047 | psexec, wmiexec |
| Defense Evasion | T1112, T1562.001 | Defender disable, log clearing |
| Impact | T1490 | File encryption (ransomware) |
| Exfiltration | T1020 | C2 exfil channel |

---

## Quick Prerequisites

Before you start:

- [ ] 5 VMs (lab network isolated, NOT connected to corporate network)
- [ ] MORPHEUS: Kali + msfconsole + hashcat + Python 3
- [ ] NEO/TRINITY/NEB/ZION: domain-joined to corp.lab
- [ ] All hosts synchronized time
- [ ] Test files created in user documents and shared folders (STEP 0 in execution plan)
- [ ] Sensitive file server shares set up (NEB)

**WARNING:** Ransomware binary will encrypt files. Do this in an **isolated lab environment only**. Snapshots recommended before STEP 5.

---

## Important Notes

⚠️ **Destructive:** This scenario **actually encrypts files** using a test ransomware binary. Do NOT run in production or non-isolated lab.

⚠️ **Lab credentials are placeholders:** Replace `<LAB_ADMIN_PASSWORD>` and `<LAB_SSH_PASSWORD>` with your actual lab values.

⚠️ **Antivirus/EDR active:** This scenario includes techniques to bypass Windows Defender and EVD detection — part of the training value, but adds complexity.

⚠️ **Time investment:** Full chain (initial access → encryption) takes 2–5 hours depending on credential cracking time.

---

## Next Steps

1. **Read [`FIN6_Execution_Plan.md`](./FIN6_Execution_Plan.md)** — start at Infrastructure Setup
2. **Create test data** — run STEP 0 (populate documents + shares)
3. **Phase 1: Enabling objectives** — run STEP 1–4 (credential access + lateral movement, ~2 hours)
4. **Phase 2: Ransomware** — run STEP 5–6 (encryption, ~30 min)

---

## References

- [CTID Adversary Emulation Library — FIN6](https://github.com/center-for-threat-informed-defense/adversary_emulation_library/tree/master/fin6)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [FIN6 Profile](https://attack.mitre.org/groups/G0037/)

---

**For Training Use Only — Authorized Personnel Only**
