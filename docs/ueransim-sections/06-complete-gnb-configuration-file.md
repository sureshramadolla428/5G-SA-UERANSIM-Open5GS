# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 6. Complete gNB Configuration File

Upstream sample: `UERANSIM/config/open5gs-gnb.yaml`. Project copy: `private-5g/ueransim/config/open5gs/open5gs-gnb.yaml`.

```yaml
mcc: '999'
mnc: '70'
nci: '0x000000010'
idLength: 32
tac: 1

linkIp: 127.0.0.1
ngapIp: 127.0.0.1
gtpIp: 127.0.0.1

amfConfigs:
  - address: 127.0.0.5      # Open5GS native AMF bind
    port: 38412             # Use 127.0.0.1 if Open5GS in Docker with published port

slices:
  - sst: 1

ignoreStreamIds: true
```

| Field | Must match |
|---|---|
| `mcc`, `mnc` | Open5GS AMF PLMN (999/70) |
| `tac` | AMF TAI (1) |
| `amfConfigs.address` | AMF NGAP IP |
| `slices` | AMF `plmn_support.s_nssai` |

**Docker Open5GS + host UERANSIM:** set `amfConfigs.address: 127.0.0.1` and ensure `38412:38412/sctp` is published.

---
