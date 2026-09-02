# QBOT + ZeroLogon Scenario: README

**Quick Facts**
- **Time:** 3–8 hours (depends on scope and lateral movement depth)
- **Tools:** Cobalt Strike, Mimikatz, QBOT malware, ZeroLogon exploit, PSView, bitsadmin
- **Scope:** Full ATT&CK chain from spear phishing → domain compromise via ZeroLogon
- **Lab:** 5+ VMs (1 attacker, 1+ initial target, 1 file server, 1+ domain controller)

---

## What Is This?

A **hands-on training scenario** that simulates a realistic QBOT + ZeroLogon intrusion chain based on DFIR report analysis:
- Initial access via spear phishing (QBOT malware in ZIP)
- Cobalt Strike beacon establishment
- Domain reconnaissance
- Credential extraction via Mimikatz
- ZeroLogon exploitation (CVE-2020-1472) for domain controller compromise
- Persistence & lateral movement
- Data exfiltration

Uses **QBOT emulation + genuine ZeroLogon exploit** to teach tactics/techniques for detecting advanced domain compromise chains.

---

## Lab Setup

| Host | OS | Role | IP | Notes |
|------|----|----|-----|--------|
| **MORPHEUS** | Kali Linux | Attacker (C2) | 172.20.43.5 | Cobalt Strike teamserver |
| **TRINITY** | Windows 11 | Initial target | 172.20.43.X | Initial QBOT compromise |
| **NEO** | Windows Server | Secondary target | 172.20.43.X | Lateral movement target |
| **NEB** | Windows Server | File server | 172.20.43.X | Data staging & exfiltration |
| **ZION** | Windows Server 2019 | Domain controller | 172.20.43.X | ZeroLogon target for domain takeover |

**Domain:** `<DOMAIN_NAME>` | **Admin:** `Administrator` / `<LAB_ADMIN_PASSWORD>`

**Pre-cached Tools:** `/home/test/Desktop/Adversary_Profiles/ZeroLogon/`

---

## Attack Flow (8 Phases)

```
PHASE 1: Reconnaissance (T1592, T1589)
└─ Information gathering on targets

PHASE 2: Weaponization (T1566, T1588)
└─ Generate malicious EXE via Cobalt Strike

PHASE 3: Delivery (T1598, T1193)
└─ Spear phishing email with ZIP attachment (QBOT)

PHASE 4: Exploitation (T1204, T1059)
└─ User extracts ZIP, executes EXE, QBOT runs

PHASE 5: Installation (T1547, T1053)
└─ Cobalt Strike beacon + persistence mechanisms

PHASE 6: Command & Control (T1087, T1018, T1083, T1003)
└─ Domain enumeration, host discovery, credential extraction

PHASE 7: Privilege Escalation (T1212, T1068)
└─ ZeroLogon exploit (CVE-2020-1472) for DC compromise

PHASE 8: Exfiltration & Impact (T1560, T1567, T1486)
└─ Archive and upload data, establish domain persistence
```

---

## ATT&CK Techniques Demonstrated

| Phase | Technique ID | What It Does |
|-------|-------------|-------------|
| Reconnaissance | T1592.004, T1589.001 | Gather email/system info on targets |
| Weaponization | T1566.001, T1588.004 | Craft malicious EXE via Cobalt Strike |
| Delivery | T1598.001, T1193 | Spear phishing email with ZIP attachment |
| Exploitation | T1204.002, T1204.003 | User executes malicious EXE |
| Installation | T1547.001, T1547.010 | Beacon establishment + persistence |
| Discovery | T1087.002, T1018, T1083, T1135 | Domain/host/share enumeration |
| Credential Access | T1003.001, T1110 | Mimikatz credential dumping |
| Lateral Movement | T1021.001, T1021.006 | RDP, WinRM to other hosts |
| Privilege Escalation | T1212, T1068 | ZeroLogon (CVE-2020-1472) to get SYSTEM on DC |
| Collection | T1560.001, T1123 | Archive & collect sensitive data |
| Exfiltration | T1567, T1048 | Upload via web service (bashupload, transfer.sh) |
| Impact | T1531 | Full domain compromise & persistence |

---

## Tools & Downloads

Before starting, download and prepare these tools on MORPHEUS:

| Tool | Purpose | Download Link |
|------|---------|---------------|
| **Cobalt Strike** | Advanced C2 framework | [Cobalt Strike](https://www.cobaltstrike.com/) |
| **Mimikatz** | Credential dumping | [GitHub Release](https://github.com/gentilkiwi/mimikatz/releases) |
| **ZeroLogon Exploit** | CVE-2020-1472 DC compromise | [SecureAuthCorp/zerologon](https://github.com/SecureAuthCorp/zerologon) |
| **Invoke-ShareFinder.ps1** | Share enumeration | [Veil-PowerView](https://github.com/darkoperator/Veil-PowerView/tree/master/PowerView/functions) |
| **NetScan Portable** | Network scanning | [Nmap](https://nmap.org/download.html) or portable alternatives |
| **AnyDesk** (optional) | Remote desktop alternative | https://anydesk.com/en/download |
| **Proton Mail** | Phishing delivery | https://account.proton.me |
| **7-Zip (portable)** | Archive tool | https://www.7-zip.org/download.html |
| **bitsadmin** | Built-in Windows tool | Pre-installed on Windows |
| **xfreerdp** | RDP client (Linux) | `apt install freerdp2-x11` |

**Pre-cached location:** `/home/test/Desktop/Adversary_Profiles/ZeroLogon/`

---

## Quick Prerequisites

Before you start:

- [ ] 5+ VMs (lab network isolated from production)
- [ ] MORPHEUS: Kali Linux with Cobalt Strike trial/full
- [ ] TRINITY/NEO/NEB: Windows 11/Server, domain-joined to `<DOMAIN_NAME>`
- [ ] ZION: Windows Server 2019 (DC running vulnerable netlogon service)
- [ ] All hosts synchronized time
- [ ] Vision One agent monitoring (or equivalent EDR)
- [ ] Pre-cached tools in `/home/test/Desktop/Adversary_Profiles/ZeroLogon/`
- [ ] Test data staged on TRINITY/NEB (sensitive documents)
- [ ] Temporary Proton Mail account ready at https://account.proton.me
- [ ] File upload service account ready (bashupload.com, transfer.sh, or similar)

**Key check:** Verify MORPHEUS can reach DC (ZION) on port 389 (NetLogon) for ZeroLogon exploit before PHASE 7.

---

## Important Notes

⚠️ **All sample data is fictional:** Employee names, emails, company data are lab-generated, not real.

⚠️ **Credentials are placeholders:** Replace `<LAB_ADMIN_PASSWORD>`, `<PROTON_EMAIL>`, `<PROTON_PASSWORD>` with your actual lab credentials — never reuse real passwords.

⚠️ **This is a training emulation:** Uses Cobalt Strike + ZeroLogon exploit to emulate QBOT TTPs; not a replica of actual QBOT malware.

⚠️ **Email account requirement:** You will need a temporary Proton Mail account. Do not use production email accounts.

⚠️ **ZeroLogon is destructive:** CVE-2020-1472 can cause domain controller instability. Only exploit in isolated lab.

⚠️ **This requires Cobalt Strike:** Trial or full license needed; not compatible with open-source alternatives (though Meterpreter can substitute in some phases).

---

## Next Steps

1. **Read [`QBOT_ZeroLogon_Execution_Plan.md`](./QBOT_ZeroLogon_Execution_Plan.md)** — start at Infrastructure Setup
2. **Set up Cobalt Strike** — download and configure teamserver + client
3. **Execute attack chain** — run PHASE 2–6 (weaponization → persistence, ~2–3 hours)
4. **ZeroLogon exploitation** (optional) — run PHASE 7 for DC compromise (1–2 hours)
5. **Exfiltration & impact** (optional/final) — run PHASE 8 for full chain (1 hour)

---

## References & Resources

**Threat Intelligence & ATT&CK**
- [DFIR Report: QBOT and ZeroLogon Lead to Full Domain Compromise](https://thedfirreport.com/2022/02/21/qbot-and-zerologon-lead-to-full-domain-compromise/)
- [DFIR Report: From Zero to Domain Admin](https://thedfirreport.com/2021/11/01/from-zero-to-domain-admin/)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [CVE-2020-1472: ZeroLogon](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-1472)

**Tools & Documentation**
- [Cobalt Strike Documentation](https://www.cobaltstrike.com/)
- [Cobalt Strike Listeners & Payloads](https://www.cobaltstrike.com/docs)
- [Mimikatz GitHub Repository](https://github.com/gentilkiwi/mimikatz)
- [ZeroLogon Exploit (SecureAuthCorp)](https://github.com/SecureAuthCorp/zerologon)
- [Invoke-ShareFinder (PowerView)](https://github.com/darkoperator/Veil-PowerView/tree/master/PowerView/functions)

**C2 & Exploitation**
- [Beacon Command Reference](https://www.cobaltstrike.com/docs)
- [PowerView: A PowerShell Post-Exploitation Framework](https://github.com/PowerShellMafia/PowerSploit)
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
