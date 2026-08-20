# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 14. Break Testing & Stress Testing

```bash
# Wrong key
cp config/open5gs-ue.yaml config/open5gs-ue-bad.yaml
sed -i 's/465B5CE8B199B49FAA5F0A2EE238A6BC/00000000000000000000000000000000/' config/open5gs-ue-bad.yaml
sudo ./build/nr-ue -c config/open5gs-ue-bad.yaml
# Expected: Authentication Reject

# Block N2
sudo iptables -I OUTPUT -p sctp --dport 38412 -j DROP
sudo iptables -D OUTPUT -p sctp --dport 38412 -j DROP

# Stop AMF while attached
docker compose stop amf
```

---
