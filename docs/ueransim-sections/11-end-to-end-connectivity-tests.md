# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 11. End-to-End Connectivity Tests

```bash
ping -I uesimtun0 -c 4 8.8.8.8
curl --interface uesimtun0 -s http://httpbin.org/ip
nslookup google.com uesimtun0 2>/dev/null || dig @8.8.8.8 google.com +short
iperf3 -c iperf.he.net -B $(ip -4 addr show uesimtun0 | grep inet | awk '{print $2}' | cut -d/ -f1) -t 5
```

---
