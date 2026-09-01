# APT29 Scenario 1

**Quick Facts**
- **Time:** 1–4 hours (depends on scope)
- **Tools:** Meterpreter, Impacket, evil-winrm
- **Scope:** Full ATT&CK chain from initial access → exfiltration
- **Lab:** 5 VMs (1 attacker, 4 targets)

---

## What Is This?

A **hands-on training scenario** that simulates a realistic APT29-style espionage intrusion:
- Initial access via macro → Meterpreter beacon
- Discovery + credential extraction
- Lateral movement to 3 additional hosts
- Data collection & exfiltration

Uses **generic red-team tooling** to teach tactics/techniques, not a replica of APT29's actual malware.

---

## Lab Setup

| Host | OS | Role | IP |
|------|----|----|-----|
| **MORPHEUS** | Kali Linux | Attacker (C2) | 192.168.150.50 |
| **TRINITY** | Windows 11 | Initial target | 192.168.150.22 |
| **NEO** | Windows 11 | Secondary target | 192.168.150.21 |
| **NEB** | Server 2025 | File server | 192.168.150.20 |
| **ZION** | Server 2025 | Domain controller | 192.168.150.10 |

**Domain:** `corp.lab` | **Admin:** `Administrator` / `<LAB_ADMIN_PASSWORD>`

---

## Attack Flow (5 Phases)

```
PHASE 1: Initial Access (T1566 + T1059)
└─ Macro execution on TRINITY → Meterpreter beacon

PHASE 2: Discovery (T1087, T1018, T1083)
└─ Domain enumeration from TRINITY beacon

PHASE 3: Credential Access (T1003, T1550)
└─ Extract domain admin hash via secretsdump

PHASE 4: Lateral Movement (T1021, T1047)
└─ psexec/wmiexec/WinRM/RDP to NEO, NEB, ZION

PHASE 5: Collection & Exfiltration (T1560, T1041)
└─ Archive & download sensitive files to MORPHEUS
```

---

## Files in This Scenario

| File | What It Has |
|------|------------|
| **README.md** | This overview (you are here) |
| **APT29_Execution_Plan.md** | Step-by-step execution (STEP 0–10) + troubleshooting |

---

## ATT&CK Techniques Demonstrated

| Phase | Technique ID | What It Does |
|-------|-------------|-------------|
| Initial Access | T1566.001 | Spearphishing macro |
| Execution | T1059.001 | PowerShell payload |
| Discovery | T1087.002, T1018, T1083, T1135 | Domain/system/share enumeration |
| Credential Access | T1003.001, T1003.003 | LSASS dump, NTDS extraction |
| Persistence | T1053.005 | Scheduled task |
| Lateral Movement | T1021.002, T1047, T1021.006, T1021.001 | psexec, wmiexec, WinRM, RDP |
| Collection | T1560.001 | Archive data |
| Exfiltration | T1041 | C2 channel |

---

## Quick Prerequisites

Before you start:

- [ ] 5 VMs (lab network isolated)
- [ ] MORPHEUS: Kali + msfconsole + Python 3
- [ ] TRINITY/NEO/NEB/ZION: domain-joined to corp.lab
- [ ] All hosts synchronized time
- [ ] Vision One agent monitoring
- [ ] Test files created (STEP 0 in plan.md)

**Key check:** Run `nmap -p 135,139,445,3389,5985,5986 -Pn 192.168.150.21` from MORPHEUS before STEP 6 — tells you which lateral movement methods will work.

---

## Important Notes

⚠️ **All sample data is fictional:** Employee names, emails, payroll data in STEP 0 are lab-generated, not real.

⚠️ **Password is a placeholder:** Replace `<LAB_ADMIN_PASSWORD>` with your actual lab credential — never reuse a real password.

⚠️ **Training purpose only:** This emulates APT29 TTPs using generic tools; it is not a fidelity replica of actual APT29 malware/infrastructure.

---

## Next Steps

1. **Read [`APT29_Execution_Plan.md`](./APT29_Execution_Plan.md)** — start at Infrastructure Setup
2. **Create test files** — run STEP 0
3. **Execute attack chain** — run STEP 1–5 (core chain, ~1–2 hours)
4. **Lateral movement** (optional) — run STEP 6–10 for additional hosts

---

## References

- [CTID Adversary Emulation Library](https://github.com/center-for-threat-informed-defense/adversary_emulation_library/tree/master/apt29)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [APT29 Profile](https://attack.mitre.org/groups/G0016/)

---

**For Training Use Only — Authorized Personnel Only**
