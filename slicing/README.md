# Network Slicing + QoS (Open5GS + UERANSIM) — tested procedure

Configure **two S-NSSAI slices** so one UE holds **two independent PDU sessions/tunnels**, each
with its own QoS (bandwidth cap). This is the exact sequence that worked, including the gotchas.

## Slice plan
| Slice | S-NSSAI | DNN | QoS (Session-AMBR) |
|-------|---------|-----|--------------------|
| A | SST 1 / SD 000001 | internet | 1 Gbps |
| B | SST 1 / SD 000002 | internet | 10 Mbps (the cap we prove) |

There are **four** places the slices must all agree: gNB, UE, AMF, and the subscriber. If any
one disagrees you get errors (`slice-not-supported` on NG Setup, or `DNN_NOT_SUPPORTED_OR_NOT_SUBSCRIBED`
on the PDU session).

## 1. gNB — `config/open5gs-gnb.yaml`
```yaml
slices:
  - sst: 1
    sd: 0x000001
  - sst: 1
    sd: 0x000002
```

## 2. UE — `config/open5gs-ue.yaml`
```yaml
sessions:
  - type: 'IPv4'
    apn: 'internet'
    slice: { sst: 1, sd: 0x000001 }
  - type: 'IPv4'
    apn: 'internet'
    slice: { sst: 1, sd: 0x000002 }
configured-nssai:
  - sst: 1
    sd: 0x000001
  - sst: 1
    sd: 0x000002
default-nssai:
  - sst: 1
    sd: 0x000001
```

## 3. AMF — teach the core both slices (this fixes `slice-not-supported`)
The AMF only accepts slices in its `plmn_support.s_nssai`. Edit `~/docker_open5gs/amf/amf.yaml`:
```bash
sed -i '/s_nssai:/{n;s/- sst: 1/- sst: 1\n            sd: 000001\n          - sst: 1\n            sd: 000002/}' ~/docker_open5gs/amf/amf.yaml
cd ~/docker_open5gs && docker compose -f sa-deploy.yaml restart amf
```
Result (`grep -A6 s_nssai ~/docker_open5gs/amf/amf.yaml`):
```yaml
        s_nssai:
          - sst: 1
            sd: 000001
          - sst: 1
            sd: 000002
```

## 4. Subscriber — two slices, each with its OWN internet session (this fixes DNN_NOT_SUBSCRIBED)
In the WebUI (`http://<vm-ip>:9999`) give subscriber `999700000000001` two slices, each with a
Session (DNN `internet`, 5QI 9, AMBR). **Gotcha:** the WebUI's first (default) slice is created
with *no SD* — it must be set to SD `000001` to match the configs. Fix via the DB:
```bash
# give the default slice SD 000001
docker exec mongo mongosh open5gs --quiet --eval 'db.subscribers.updateOne({imsi:"999700000000001","slice.default_indicator":true},{$set:{"slice.$.sd":"000001"}})'
# confirm both slices have SDs
docker exec mongo mongosh open5gs --quiet --eval 'JSON.stringify(db.subscribers.findOne({imsi:"999700000000001"}).slice.map(s=>({sst:s.sst,sd:s.sd})))'
# -> [{"sst":1,"sd":"000001"},{"sst":1,"sd":"000002"}]
```
Note: the main SMF has no `info:` block, so it doesn't restrict by slice — the DNN subscription
lives entirely in the subscriber. Each slice needs its own `internet` session.

## 5. Run and verify TWO tunnels
```bash
cd ~/private-5g/UERANSIM
sudo killall -9 nr-ue nr-gnb 2>/dev/null
./build/nr-gnb -c config/open5gs-gnb.yaml        # Terminal 1 -> "NG Setup procedure is successful"
sudo ./build/nr-ue -c config/open5gs-ue.yaml     # Terminal 2 -> TWO "PDU Session establishment is successful"
ip -brief addr show | grep uesimtun              # Terminal 3 -> uesimtun0 AND uesimtun1
```

## 6. QoS — prove the per-slice bandwidth cap
Set Slice B to 10 **Mbps** (unit 2 = Mbps; the WebUI often defaults to Gbps = unit 3):
```bash
docker exec mongo mongosh open5gs --quiet --eval 'db.subscribers.updateOne({imsi:"999700000000001","slice.sd":"000002"},{$set:{"slice.$.session.0.ambr.downlink.value":10,"slice.$.session.0.ambr.downlink.unit":2,"slice.$.session.0.ambr.uplink.value":10,"slice.$.session.0.ambr.uplink.unit":2}})'
```
Restart the UE (so the new AMBR applies), then compare throughput per tunnel:
```bash
curl --interface uesimtun0 -o /dev/null -w "Slice A (uesimtun0): %{speed_download} bytes/s\n" http://speedtest.tele2.net/10MB.zip
curl --interface uesimtun1 -o /dev/null -w "Slice B (uesimtun1): %{speed_download} bytes/s\n" http://speedtest.tele2.net/10MB.zip
```
Slice B should top out near **1,250,000 bytes/s (~10 Mbps)** while Slice A runs much faster — the
UPF enforcing the per-slice Session-AMBR. That is network slicing + QoS demonstrated end to end.

## Troubleshooting recap
| Error | Cause | Fix |
|-------|-------|-----|
| NG Setup `slice-not-supported` | AMF doesn't list the slice | Step 3 (AMF `s_nssai`) |
| PDU `DNN_NOT_SUPPORTED_OR_NOT_SUBSCRIBED` | Subscriber slice lacks the DNN, or SD mismatch | Step 4 (subscriber slice + SD) |
| Only one `uesimtun` | The other slice's SD/DNN doesn't match across gNB/UE/AMF/subscriber | Make all four agree |

## Results & gotchas (from a real run)
- **The cap must be below the link's real throughput to be visible.** On a VM whose actual
  internet is ~4 Mbps, a 10 Mbps or 1 Gbps AMBR never bites — both slices just run at ~4 Mbps.
  Set the capped slice *below* the real rate (e.g. 1 Mbps) to see the difference.
- **Verified result:** with Slice B capped at 1 Mbps, one tunnel measured ~15 KB/s (throttled,
  timed out) while the other ran at ~500 KB/s (~4 Mbps) — ~30x difference = per-slice QoS enforced.
- **Tunnel numbering is by PDU-session completion order, not config order.** So `uesimtun0` is
  not guaranteed to be Slice A — check the UE log (`TUN interface[uesimtunX ...]` next to each
  PSI/slice) to map tunnel -> slice definitively. The throughput difference is the real proof.
- **Keep the gNB running in its own terminal** the whole time; if it exits, the UE reports
  "no cells in coverage".
- Use `curl --max-time 25` so a throttled download doesn't hang the test.
