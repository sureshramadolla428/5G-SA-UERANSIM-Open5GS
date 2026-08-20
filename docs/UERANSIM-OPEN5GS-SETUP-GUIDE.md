# UERANSIM + Open5GS 5G Core — Complete Setup, Test & Stress Guide

> **Audience:** Linux administrators building a software-only 5G lab  
> **Target OS:** Ubuntu 22.04+ inside VMware  
> **Scope:** Build UERANSIM, configure gNB/UE for Open5GS, verify, capture traffic, multi-UE load, break-test

---

## About This Guide

This guide walks through **UERANSIM** — open-source 5G gNB and UE simulator — connected to **Open5GS** 5G Core. It covers source builds, configuration aligned with upstream `config/open5gs-gnb.yaml` and `config/open5gs-ue.yaml`, signaling flows, PCAP analysis, and fault injection.

**Pre-built configs in this project:**

- `private-5g/ueransim/config/open5gs/open5gs-gnb.yaml`
- `private-5g/ueransim/config/open5gs/open5gs-ue.yaml`

**Default PLMN:** MCC **999** / MNC **70** (Open5GS + UERANSIM standard test network)

---

## Table of Contents

1. [What Is UERANSIM and Why Use It](#1-what-is-ueransim-and-why-use-it)
2. [Complete 5G Signaling Flow Diagrams](#2-complete-5g-signaling-flow-diagrams)
3. [Prerequisites & System Requirements](#3-prerequisites--system-requirements)
4. [Install All Build Dependencies](#4-install-all-build-dependencies)
5. [Clone and Build UERANSIM](#5-clone-and-build-ueransim)
6. [Complete gNB Configuration File](#6-complete-gnb-configuration-file)
7. [Complete UE Configuration File](#7-complete-ue-configuration-file)
8. [Provision the Subscriber in Open5GS](#8-provision-the-subscriber-in-open5gs)
9. [Start and Verify the gNB](#9-start-and-verify-the-gnb)
10. [Start and Verify the UE](#10-start-and-verify-the-ue)
11. [End-to-End Connectivity Tests](#11-end-to-end-connectivity-tests)
12. [PCAP Capture for gNB and UE Traffic](#12-pcap-capture-for-gnb-and-ue-traffic)
13. [Multiple UE Simulation](#13-multiple-ue-simulation)
14. [Break Testing & Stress Testing](#14-break-testing--stress-testing)
15. [Troubleshooting Common Issues](#15-troubleshooting-common-issues)

---

## 1. What Is UERANSIM and Why Use It

**UERANSIM** ([github.com/aligungr/UERANSIM](https://github.com/aligungr/UERANSIM)) implements:

- **nr-gnb** — 5G gNodeB (base station simulator)
- **nr-ue** — 5G UE (phone simulator)

The radio stack runs over **UDP** (not real RF). N2/N3 connect to Open5GS like a real RAN.

When paired with **Open5GS**, you get a complete SA 5G network on Linux.

### 1.1 Components

| Binary | Role |
|---|---|
| `nr-gnb` | gNB — NGAP to AMF, GTP-U to UPF |
| `nr-ue` | UE — NAS registration, creates `uesimtun0` |

### 1.2 Where UERANSIM Fits with Open5GS

```text
  ┌──────── UERANSIM ────────┐          ┌──── Open5GS ──────┐
  │  nr-ue  ←RRC→  nr-gnb    │          │ AMF SMF UPF UDM …   │
  │     │              │     │  N2 SCTP │                     │
  │ uesimtun0            └────┼─────────►│ :38412              │
  │                            │  N3 GTP  │ :2152               │
  └────────────────────────────┘          └─────────────────────┘
```

---

## 2. Complete 5G Signaling Flow Diagrams

### 2.1 Registration + PDU Session

```text
  UE          gNB         AMF         AUSF/UDM       SMF         UPF
   │──Registration──►│──NGAP──────►│──SBI auth──►│              │
   │◄──Accept────────│◄───────────│◄────────────│              │
   │──PDU Session───►│──NGAP──────►│────────────►│──PFCP──────►│
   │◄──IP assigned───│◄───────────│◄────────────│◄────────────│
   │════════ GTP-U user traffic via uesimtun0 ═══════════════════►
```

---

## 3. Prerequisites & System Requirements

| Requirement | Minimum |
|---|---|
| OS | Ubuntu 22.04+ (VMware guest) |
| RAM | 4 GB (+ Open5GS RAM) |
| CPU | 2 cores |
| Open5GS | Running (see OPEN5GS-CORE-SETUP-GUIDE.md) |
| SCTP | `modprobe sctp` |

---

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

## 9. Start and Verify the gNB

**Terminal 1 — start Open5GS first** (see core guide).

**Terminal 2 — gNB:**

```bash
cd /private-5g/UERANSIM/build
./nr-gnb -c ../config/open5gs-gnb.yaml
```

Expected log lines:

```text
NG Setup procedure is successful
```

| Error | Cause | Fix |
|---|---|---|
| `Connection refused` | AMF not listening | Start Open5GS; check port 38412 |
| `NGAP setup failure` | PLMN mismatch | Use 999/70 everywhere |
| SCTP timeout | Firewall | Allow SCTP egress |

```bash
# Verify from another terminal
sudo ss -lnp | grep 38412
docker compose logs amf 2>/dev/null | grep -iE "NGSetup|gNB"
```

---

## 10. Start and Verify the UE

**Terminal 3 — UE (requires root for TUN):**

```bash
cd /private-5g/UERANSIM/build
sudo ./nr-ue -c ../config/open5gs-ue.yaml
```

Expected:

```text
Registration accept
PDU Session establishment accept
```

```bash
ip addr show uesimtun0
# inet 10.45.x.x/32 or similar from Open5GS pool
```

---

## 11. End-to-End Connectivity Tests

```bash
ping -I uesimtun0 -c 4 8.8.8.8
curl --interface uesimtun0 -s http://httpbin.org/ip
nslookup google.com uesimtun0 2>/dev/null || dig @8.8.8.8 google.com +short
iperf3 -c iperf.he.net -B $(ip -4 addr show uesimtun0 | grep inet | awk '{print $2}' | cut -d/ -f1) -t 5
```

---

## 12. PCAP Capture for gNB and UE Traffic

```bash
sudo tcpdump -i any 'sctp port 38412 or udp port 2152 or host 127.0.0.1' -w ueransim-open5gs.pcap
```

Filter in Wireshark: `ngap`, `pfcp`, `gtp`.

---

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

## 15. Troubleshooting Common Issues

| Symptom | Fix |
|---|---|
| SCTP refused | Publish AMF port 38412; start amf container |
| Auth reject | Match K/OPc in WebUI and ue.yaml |
| No uesimtun0 IP | Add DNN `internet` in WebUI; check SMF/UPF logs |
| UE can't reach gNB | Align `gnbSearchList` with gNB `linkIp` |
| PLMN mismatch | Use 999/70 on gNB, UE, and Open5GS AMF |

### Registration Accept crash (GCC 15)

**Symptom:** UE crashes immediately after `Registration accept received`, with a `stl_vector.h` assertion referencing `Plmn`.

**Cause:** UERANSIM v3.3.0 has a bug in `NasList` (`src/lib/nas/storage.hpp`) that triggers vector bounds checks under GCC 15 when clearing or shifting list entries.

**Fix:** apply the NasList patch from the **private scripts** companion (not in this public pack) and rebuild UERANSIM:

```bash
cd ~/UERANSIM
patch -p1 < /path/to/ueransim-naslist-fix.patch
make clean
make -j$(nproc)
```

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
