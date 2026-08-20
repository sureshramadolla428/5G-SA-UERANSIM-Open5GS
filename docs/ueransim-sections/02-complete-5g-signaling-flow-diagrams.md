# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 2. Complete 5G Signaling Flow Diagrams

### 2.1 Registration + PDU Session

```text
  UE          gNB         AMF         AUSF/UDM       SMF         UPF
   │──Registration──►│──NGAP──────►│──SBI auth──►│              │
   │◄──Accept────────│◄───────────│◄────────────│              │
   │──PDU Session───►│──NGAP──────►│────────────►│──PFCP──────►│
   │◄──IP assigned───│◄───────────│◄────────────│◄────────────│
   │════════ GTP-U user traffic via uesimtun0 ═══════════════════►
```

---
