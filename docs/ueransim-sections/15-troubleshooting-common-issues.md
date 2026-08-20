# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 15. Troubleshooting Common Issues

| Symptom | Fix |
|---|---|
| SCTP refused | Publish AMF port 38412; start amf container |
| Auth reject | Match K/OPc in WebUI and ue.yaml |
| No uesimtun0 IP | Add DNN `internet` in WebUI; check SMF/UPF logs |
| UE can't reach gNB | Align `gnbSearchList` with gNB `linkIp` |
| PLMN mismatch | Use 999/70 on gNB, UE, and Open5GS AMF |

---

## Quick Reference

```bash
# Build
cd /private-5g/UERANSIM && make -j$(nproc)

# Run (Open5GS must be up; subscriber provisioned)
./build/nr-gnb -c config/open5gs-gnb.yaml &
sudo ./build/nr-ue -c config/open5gs-ue.yaml

# Test
ping -I uesimtun0 -c 4 8.8.8.8
```

| Parameter | Value |
|---|---|
| PLMN | 999 / 70 |
| IMSI | 999700000000001 |
| AMF (native) | 127.0.0.5:38412 |
| AMF (Docker) | 127.0.0.1:38412 |
| WebUI | http://127.0.0.1:9999 |
| DNN | internet |
| UE IP pool | 10.45.0.0/16 |

---

*Document version: 2.0 — UERANSIM + Open5GS Setup Guide (replaces UERANSIM-ELLA-5G-SETUP-GUIDE.md)*
