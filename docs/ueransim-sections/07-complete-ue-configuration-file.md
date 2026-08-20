# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 7. Complete UE Configuration File

Upstream: `UERANSIM/config/open5gs-ue.yaml`. Project copy: `private-5g/ueransim/config/open5gs/open5gs-ue.yaml`.

```yaml
supi: 'imsi-999700000000001'
mcc: '999'
mnc: '70'
key: '465B5CE8B199B49FAA5F0A2EE238A6BC'
op: 'E8ED289DEBA952E4283B54E88E6183CA'
opType: 'OPC'
amf: '8000'

gnbSearchList:
  - 127.0.0.1

sessions:
  - type: 'IPv4'
    apn: 'internet'
    slice:
      sst: 1
```

| Field | Must match |
|---|---|
| `supi` / IMSI | Open5GS WebUI subscriber |
| `key`, `op`, `opType` | WebUI K, OPc |
| `sessions[].apn` | SMF DNN `internet` |
| `gnbSearchList` | gNB `linkIp` |

---
