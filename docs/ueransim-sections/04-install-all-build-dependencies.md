# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 4. Install All Build Dependencies

```bash
sudo apt update
sudo apt install -y \
  build-essential cmake make g++ gcc \
  libsctp-dev lksctp-tools \
  git iproute2

# cmake via snap if apt version too old
sudo snap install cmake --classic 2>/dev/null || true
```

---
