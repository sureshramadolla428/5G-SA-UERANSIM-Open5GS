# 5G SA Lab: Open5GS Core + UERANSIM gNB/UE

### Terrestrial UE â†” gNB â†” 5G core (software lab, PLMN 999/70)

![5G SA](https://img.shields.io/badge/3GPP-5G%20SA-0033A0)
![Open5GS](https://img.shields.io/badge/Open5GS-core-1F4E79)
![UERANSIM](https://img.shields.io/badge/UERANSIM-gNB%20%2B%20UE-0A66C2)
This repository is a **GitHub-ready public pack** for a **terrestrial 5G Standalone** lab:

- **Core:** Open5GS (AMF, SMF, UPF, NRF, â€¦) via Docker Compose or Kubernetes/Helm
- **RAN:** UERANSIM **gNB**
- **UE:** UERANSIM **UE** (`uesimtun0` user plane)

It is **not** NTN, **not** ATG/A2G, and it does **not** include offline RAG, ML, or career-profile material.

**Success check:** `ping -I uesimtun0 -c 4 8.8.8.8` with 0% loss.

---

## Scope

| Included | Not included |
|----------|----------------|
| Open5GS 5G SA core Compose file + lab `.env` | NTN / LEO / SIB19 |
| UERANSIM gNB + UE YAML (PLMN 999/70) | ATG / aircraft kinematics |
| Kubernetes CNF install (Gradiant charts) | Offline 3GPP RAG |
| Network slicing notes (S-NSSAI / Session-AMBR) | Grafana/ML/pcap-monitor stacks |
| Bring-up, ping, multi-UE, break-test docs | Captures, logs, 3GPP PDFs |

Open5GS and UERANSIM are **upstream** projects. This pack vendors **lab configs and runbooks**, not those source trees.

---

## Layout

```
github-ueransim-open5gs/
â”œâ”€â”€ docker-compose/          # Open5GS SA core (sa-deploy.yaml + .env)
â”œâ”€â”€ ueransim/config/         # open5gs-gnb.yaml, open5gs-ue.yaml
â”œâ”€â”€ kubernetes/              # Helm CNF notes (public helm commands)
â”œâ”€â”€ slicing/                 # two S-NSSAI slices + QoS procedure
â”œâ”€â”€ docs/                    # Docker / k8s workflows + UERANSIM guide
â”œâ”€â”€ NOTICE                   # upstream license references
â””â”€â”€ README.md
```

---

## Quick start (Docker Compose, Ubuntu VM)

### 1. Core

```bash
cd docker-compose
docker compose -f sa-deploy.yaml up -d
docker compose -f sa-deploy.yaml ps
```

WebUI: `http://127.0.0.1:9999` (Open5GS test login `admin` / `1423`).

Add subscriber matching `ueransim/config/open5gs-ue.yaml`:

| Field | Lab value |
|-------|-----------|
| IMSI | `999700000000001` |
| K | `465B5CE8B199B49FAA5F0A2EE238A6BC` |
| OPc | `E8ED289DEBA952E4283B54E88E6183CA` |
| DNN | `internet` |

These are **public Open5GS/UERANSIM test credentials**, not production secrets.

### 2. Build UERANSIM (once)

Follow `docs/UERANSIM-OPEN5GS-SETUP-GUIDE.md` (clone + build `nr-gnb` / `nr-ue`). Copy this packâ€™s YAML into your UERANSIM `config/` directory.

On Docker labs, set **gNB `gtpIp` to the VMâ€™s IP** (not always `127.0.0.1`) so GTP-U reaches `UPF_ADVERTISE_IP=172.22.0.8`. `ngapIp` stays `127.0.0.1` when AMF is published on the host.

### 3. RAN + UE (separate terminals)

```bash
./build/nr-gnb -c config/open5gs-gnb.yaml          # Terminal 1 â€” expect NG Setup successful
sudo ./build/nr-ue -c config/open5gs-ue.yaml       # Terminal 2 â€” expect uesimtun0
ping -I uesimtun0 -c 4 8.8.8.8                      # Terminal 3
```

### 4. Stop (reverse order)

```bash
sudo killall -9 nr-ue nr-gnb 2>/dev/null
cd docker-compose && docker compose -f sa-deploy.yaml stop
```

---

## Kubernetes (optional)

Core + UERANSIM as CNFs on k3s via Gradiant Helm charts:

```bash
# discover chart versions, then install core + RAN (see kubernetes/README.md)
O5_VER=$(helm show chart oci://registry-1.docker.io/gradiantcharts/open5gs | awk '/^version:/{print $2}')
helm install open5gs oci://registry-1.docker.io/gradiantcharts/open5gs \
  --version "$O5_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/5gSA-values.yaml
RAN_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-gnb | awk '/^version:/{print $2}')
helm install ueransim-gnb oci://registry-1.docker.io/gradiant/ueransim-gnb \
  --version "$RAN_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/gnb-ues-values.yaml
kubectl -n open5gs get pods
kubectl -n open5gs exec -ti deployment/ueransim-gnb-ues -- ping -I uesimtun0 8.8.8.8
```

Details: `kubernetes/README.md` and `docs/WORKFLOW-Kubernetes.md`.

---

## Docs

| Doc | Use |
|-----|-----|
| `docs/WORKFLOW-Docker.md` | Ordered Docker bring-up |
| `docs/WORKFLOW-Kubernetes.md` | Ordered Helm/CNF bring-up |
| `docs/UERANSIM-OPEN5GS-SETUP-GUIDE.md` | Full gNB/UE build and test |
| `docs/ueransim-sections/` | Same guide split by section |
| `slicing/README.md` | Two slices on one UE |

---

## Licenses (upstream)

This repository is **configuration and documentation only**. It does not relicense Open5GS or UERANSIM.

| Project | License | Reference |
|---------|---------|-----------|
| Open5GS | AGPL-3.0 | https://github.com/open5gs/open5gs/blob/main/LICENSE |
| UERANSIM | AGPL-3.0 | https://github.com/aligungr/UERANSIM/blob/master/LICENSE |
| Gradiant 5G charts | Apache-2.0 | https://github.com/Gradiant/5g-charts/blob/main/LICENSE |

See `NOTICE`. Helper scripts (Helm install wrapper, NAS list patch) live in the **private** companion [`5G-SA-UERANSIM-Open5GS-scripts`](https://github.com/sureshramadolla428/5G-SA-UERANSIM-Open5GS-scripts), not here.

## Author

Lab configuration notes by **Suresh Ramadolla**. Open5GS and UERANSIM remain the work of their respective authors.

## License / Rights

All Rights Reserved. This public repository is a showcase; see LICENSE. No permission is granted to use, copy, modify, or distribute any part of this repository without prior written consent.
