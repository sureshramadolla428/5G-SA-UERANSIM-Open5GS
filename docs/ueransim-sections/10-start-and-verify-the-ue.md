# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 10. Start and Verify the UE

**Terminal 3 — UE (requires root for TUN):**

```bash
cd /private-5g/UERANSIM/build
sudo ./nr-ue -c ../config/open5gs-ue.yaml
```

Expected:

```text
Registration accept
PDU Session establishment accept
```

```bash
ip addr show uesimtun0
# inet 10.45.x.x/32 or similar from Open5GS pool
```

---
