# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 9. Start and Verify the gNB

**Terminal 1 — start Open5GS first** (see core guide).

**Terminal 2 — gNB:**

```bash
cd /private-5g/UERANSIM/build
./nr-gnb -c ../config/open5gs-gnb.yaml
```

Expected log lines:

```text
NG Setup procedure is successful
```

| Error | Cause | Fix |
|---|---|---|
| `Connection refused` | AMF not listening | Start Open5GS; check port 38412 |
| `NGAP setup failure` | PLMN mismatch | Use 999/70 everywhere |
| SCTP timeout | Firewall | Allow SCTP egress |

```bash
# Verify from another terminal
sudo ss -lnp | grep 38412
docker compose logs amf 2>/dev/null | grep -iE "NGSetup|gNB"
```

---
