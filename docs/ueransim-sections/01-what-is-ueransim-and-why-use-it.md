# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 1. What Is UERANSIM and Why Use It

**UERANSIM** ([github.com/aligungr/UERANSIM](https://github.com/aligungr/UERANSIM)) implements:

- **nr-gnb** — 5G gNodeB (base station simulator)
- **nr-ue** — 5G UE (phone simulator)

The radio stack runs over **UDP** (not real RF). N2/N3 connect to Open5GS like a real RAN.

When paired with **Open5GS**, you get a complete SA 5G network on Linux.

### 1.1 Components

| Binary | Role |
|---|---|
| `nr-gnb` | gNB — NGAP to AMF, GTP-U to UPF |
| `nr-ue` | UE — NAS registration, creates `uesimtun0` |

### 1.2 Where UERANSIM Fits with Open5GS

```text
  ┌──────── UERANSIM ────────┐          ┌──── Open5GS ──────┐
  │  nr-ue  ←RRC→  nr-gnb    │          │ AMF SMF UPF UDM …   │
  │     │              │     │  N2 SCTP │                     │
  │ uesimtun0            └────┼─────────►│ :38412              │
  │                            │  N3 GTP  │ :2152               │
  └────────────────────────────┘          └─────────────────────┘
```

---
