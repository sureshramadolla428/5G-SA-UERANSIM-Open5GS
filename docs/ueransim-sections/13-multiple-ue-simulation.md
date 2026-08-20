# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 13. Multiple UE Simulation

```bash
for i in $(seq 1 10); do
  cp config/open5gs-ue.yaml config/open5gs-ue-${i}.yaml
  MSIN=$(printf "%09d" $i)
  sed -i "s/999700000000001/9997000000000${i}/" config/open5gs-ue-${i}.yaml
  sed -i "s/imsi-999700000000001/imsi-9997000000000${i}/" config/open5gs-ue-${i}.yaml
done
```

Provision each IMSI in Open5GS WebUI, then:

```bash
for i in $(seq 1 10); do
  sudo ./build/nr-ue -c config/open5gs-ue-${i}.yaml > /tmp/ue-${i}.log 2>&1 &
  sleep 0.5
done
ip addr | grep uesimtun
```

---
