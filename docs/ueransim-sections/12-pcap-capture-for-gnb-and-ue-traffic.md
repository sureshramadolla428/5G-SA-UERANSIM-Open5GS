# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 12. PCAP Capture for gNB and UE Traffic

```bash
sudo tcpdump -i any 'sctp port 38412 or udp port 2152 or host 127.0.0.1' -w ueransim-open5gs.pcap
```

Filter in Wireshark: `ngap`, `pfcp`, `gtp`.

---
