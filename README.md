# Adversary Emulation Scenarios

Sanitized adversary emulation scenarios I designed to generate synthetic attack telemetry for hands-on threat hunting and incident response training. These were originally built to help new-hire analysts practice live investigation and detection workflows in a safe, controlled environment — no real customer data, environments, or proprietary tooling is represented here.

## Purpose

Reading about an attack technique and investigating one live are very different skills. These scenarios close that gap by:

- Simulating realistic adversary behavior mapped to [MITRE ATT&CK](https://attack.mitre.org/)
- Generating telemetry that mirrors what an analyst would see during a real engagement
- Giving trainees a repeatable environment to practice triage, pivoting, and hypothesis-driven hunting
- Reinforcing detection engineering by pairing each scenario with example detection logic

## What's Inside

Each scenario includes:

- **Objective** — what technique(s) or attacker behavior the scenario emulates
- **ATT&CK mapping** — tactics and techniques covered
- **Execution plan** — step-by-step attack narrative, using open tooling (e.g., Atomic Red Team, Caldera) rather than any proprietary or internal tooling
- **Expected detections** — generalized detection logic / what an analyst should look for, without vendor-specific configuration details

## Repository Structure

```
adversary-emulation-scenarios/
├── scenarios/
│   ├── scenario-01-.../
│   │   ├── README.md        # objective, ATT&CK mapping, prerequisites
│   │   ├── plan.md          # attack narrative / execution steps
│   │   └── detections.md    # generalized detection guidance
│   └── scenario-02-.../
├── templates/                # reusable scenario/report templates
└── LICENSE
```

## Disclaimer

These scenarios are generalized training material and do not represent any specific customer environment, incident, or proprietary tooling. They are shared for educational purposes to support analyst training and the broader security community.

## About Me

I'm a senior incident responder and threat researcher with 16+ years leading end-to-end IR engagements, mentoring analysts, and building AI-assisted investigative tooling.
