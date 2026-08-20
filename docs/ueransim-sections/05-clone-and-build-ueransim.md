# UERANSIM + Open5GS — Setup Guide
> Part of the complete guide. See `UERANSIM-OPEN5GS-SETUP-GUIDE.md` for the full document.

## 5. Clone and Build UERANSIM

```bash
cd /private-5g
git clone https://github.com/aligungr/UERANSIM.git
cd UERANSIM
make -j$(nproc)

ls -la build/nr-gnb build/nr-ue
```

Expected: two executables in `build/`.

Copy project configs:

```bash
mkdir -p config/open5gs
cp /path/to/private-5g/ueransim/config/open5gs/*.yaml config/open5gs/
```

---
