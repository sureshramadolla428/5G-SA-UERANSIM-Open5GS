# Docker Compose deployment

This folder holds the working Docker Compose deployment of the Open5GS 5G SA core,
as used in the lab.

## Files

| File | What it is |
|------|-----------|
| `sa-deploy.yaml` | The Compose definition for the full 5G SA core. **Includes the fix** that stops the UPF from publishing host UDP 2152, so the native UERANSIM gNB can bind GTP-U. |
| `.env` | Environment values, including `UPF_ADVERTISE_IP=172.22.0.8` (so the gNB reaches the UPF over the container network) and the PLMN (`MCC=999`, `MNC=70`). Standard Open5GS **test** values — no real secrets. |

> RAN configs for this pack live in `../ueransim/config/`. On Docker labs set gNB `gtpIp` to the
> VM IP so GTP-U reaches `UPF_ADVERTISE_IP`; `ngapIp` is typically `127.0.0.1`.

## Deploy (on the Ubuntu VM)

These files mirror `~/docker_open5gs/`. To run the core from a clone of this repo:

```bash
# from this folder on the VM
docker compose -f sa-deploy.yaml up -d
docker compose -f sa-deploy.yaml ps        # all NFs should be Up
```

Then start the RAN (UERANSIM) with this pack’s YAML (copy into your UERANSIM `config/`):

```bash
./build/nr-gnb -c config/open5gs-gnb.yaml           # Terminal 1
sudo ./build/nr-ue -c config/open5gs-ue.yaml        # Terminal 2
ping -I uesimtun0 -c 4 8.8.8.8                       # Terminal 3
```

## Key fixes captured in these files
- UPF host port 2152 no longer published (avoids "Address already in use" on the gNB).
- `UPF_ADVERTISE_IP=172.22.0.8` so N3/GTP-U is reachable over the Docker network.
- gNB `gtpIp` = VM IP, `ngapIp` = 127.0.0.1 (set in the UERANSIM config, not here).
