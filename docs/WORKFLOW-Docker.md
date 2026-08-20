% 5G SA Lab — Docker Workflow (Complete, Step by Step)
% Run the 5G core and radio with Docker Compose, in order, with the files to use

# How to use this workflow

This is the complete Docker-based path from an empty machine to a working 5G network, plus exercises and monitoring. Do the phases in order. Each phase lists **what you do**, the **files to use**, the **guide** for full detail, and **what you end with**. Everything runs on the **Ubuntu VM**.

## Files this workflow uses

| File / folder | Role |
|---------------|------|
| `docker-compose/sa-deploy.yaml` | The 5G core definition (with the UPF port-2152 fix) |
| `docker-compose/.env` | Settings: `UPF_ADVERTISE_IP`, PLMN (999/70) |
| `~/private-5g/UERANSIM/config/open5gs-gnb.yaml` | The tower (gNB) config (`gtpIp` = VM IP, `ngapIp` = 127.0.0.1) |
| `~/private-5g/UERANSIM/config/open5gs-ue.yaml` | The phone (UE) config (IMSI/key/opc) |
| `docs/UERANSIM-OPEN5GS-SETUP-GUIDE.md` | Full gNB/UE build and test |

\newpage

# Phase 0 — Environment (one time)

**Do:** Ubuntu VM (VMware, NAT), install Docker, build UERANSIM.
**Guide:** `docs/5G_Lab_Guide_updated.docx` Steps 1–6 (use the Detailed guide for every command).
**End with:** Docker installed and `nr-gnb`/`nr-ue` built.

# Phase 1 — Start the 5G core

**Files:** `docker-compose/sa-deploy.yaml`, `docker-compose/.env`
```bash
cd docker-compose        # (or ~/docker_open5gs on the VM)
docker compose -f sa-deploy.yaml up -d
docker compose -f sa-deploy.yaml ps      # all NFs should be Up
```
**End with:** the core running (amf, smf, upf, mongo, nrf, webui, …).

# Phase 2 — Add a subscriber

**File/UI:** WebUI at http://127.0.0.1:9999 (admin / 1423)
Add IMSI `999700000000001`, K `465B5CE8B199B49FAA5F0A2EE238A6BC`, OPc `E8ED289DEBA952E4283B54E88E6183CA`, DNN `internet`.
**End with:** a SIM the core will admit.

# Phase 3 — Start the radio (gNB then UE)

**Files:** the two UERANSIM configs.
```bash
# Terminal 1 — tower (leave running)
cd ~/private-5g/UERANSIM && ./build/nr-gnb -c config/open5gs-gnb.yaml
# Terminal 2 — phone (leave running)
sudo ./build/nr-ue -c config/open5gs-ue.yaml
```
**End with:** "NG Setup successful" (tower), registration + `uesimtun0` (phone).

# Phase 4 — Verify end to end

```bash
# Terminal 3
ping -I uesimtun0 -c 2 192.168.100.1     # gateway first
ping -I uesimtun0 -c 4 8.8.8.8           # internet
curl --interface uesimtun0 http://httpbin.org/ip
```
**Guide:** `docs/5G_Lab_Command_Practice_Detailed.docx` Session 1 (which terminal / why).
**End with:** 0% packet loss — the base lab works.

# Phase 5 — Multiple phones

Add subscribers `...002`/`...003` in the WebUI, then:
```bash
sudo killall -9 nr-ue 2>/dev/null
sudo ./build/nr-ue -c config/open5gs-ue.yaml -n 3 -t 1000
ip -brief addr show | grep uesimtun      # uesimtun0/1/2
```
**Guide:** Practice workbook, Session 2.

# Phase 6 — Optional packet capture

Start the capture **before** attaching a UE (N2 SCTP 38412, N3 GTP-U 2152):
```bash
sudo tcpdump -i any -n '(sctp port 38412) or (udp port 2152)' -w /tmp/sa-lab.pcap
```

# Phase 7 — Break / stress tests

```bash
sed "s/^key:.*/key: '00000000000000000000000000000000'/" config/open5gs-ue.yaml > /tmp/bad.yaml
sudo ./build/nr-ue -c /tmp/bad.yaml                              # wrong key -> MAC failure
docker compose -f sa-deploy.yaml stop amf   # control-plane outage (start to recover)
docker compose -f sa-deploy.yaml stop upf   # data-plane outage (start to recover)
```

# Phase 8 — Stop / clean up (reverse order)

```bash
sudo killall -9 nr-gnb nr-ue 2>/dev/null
docker compose -f sa-deploy.yaml stop
```

# Whole Docker path as one command sequence

```bash
# core
cd docker-compose && docker compose -f sa-deploy.yaml up -d
# subscriber via WebUI (http://127.0.0.1:9999)
# radio (after building UERANSIM)
./build/nr-gnb -c config/open5gs-gnb.yaml          # T1
sudo ./build/nr-ue -c config/open5gs-ue.yaml       # T2
ping -I uesimtun0 -c 4 8.8.8.8                      # T3
```

# If something breaks
See `docs/UERANSIM-OPEN5GS-SETUP-GUIDE.md` and `docs/ueransim-sections/15-troubleshooting-common-issues.md`.
