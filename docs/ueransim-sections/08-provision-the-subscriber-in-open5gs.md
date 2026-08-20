# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 8. Provision the Subscriber in Open5GS

### 8.1 WebUI (Recommended)

1. Ensure Open5GS + MongoDB + WebUI running
2. Open `http://127.0.0.1:9999`
3. Login: **admin** / **1423** (change after first login)
4. **Subscriber → Add:**

| Field | Value |
|---|---|
| IMSI | `999700000000001` |
| K | `465B5CE8B199B49FAA5F0A2EE238A6BC` |
| OPc | `E8ED289DEBA952E4283B54E88E6183CA` |
| AMF | `8000` |
| DNN/APN | `internet` |
| SST | `1` |

### 8.2 Verify in MongoDB (optional)

```bash
docker exec -it open5gs-mongodb mongosh open5gs --eval \
  'db.subscribers.find({imsi:"999700000000001"}).pretty()'
```

---
