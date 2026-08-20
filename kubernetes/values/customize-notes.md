# Customizing the Helm values

The Helm install uses Gradiant's upstream values files directly (proven-good).
To align the deployment with THIS lab's identifiers (PLMN 999/70, your IMSI/keys),
download the upstream files, edit the documented fields, and pass your local copies.

## 1. Get the authoritative default values (the full schema)
```bash
helm show values oci://registry-1.docker.io/gradiantcharts/open5gs --version 2.2.9 > open5gs-defaults.yaml
helm show values oci://registry-1.docker.io/gradiant/ueransim-gnb  --version 0.2.6 > ueransim-defaults.yaml
```

## 2. Get the tutorial override files (start from these)
```bash
curl -O https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/5gSA-values.yaml
curl -O https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/gnb-ues-values.yaml
```

## 3. Fields to align with this lab
In `5gSA-values.yaml` (Open5GS core):
- The `populate` / `initCommands` section registers subscribers via `open5gs-dbctl`.
  Set the subscriber to match your lab if you want consistency:
  - imsi: `999700000000001`
  - key:  `465B5CE8B199B49FAA5F0A2EE238A6BC`
  - opc:  `E8ED289DEBA952E4283B54E88E6183CA`
  - apn:  `internet`, sst: `1`, sd: (as configured)
- (Optional) `upf.containerSecurityContext.runAsUser/runAsGroup: 0` to allow tcpdump in the UPF pod.

In `gnb-ues-values.yaml` (RAN):
- `mcc: "999"`, `mnc: "70"` — must match the core and the registered subscriber.
- `sst`, `sd`, `tac` — must match the core.
- `ues.initialMSISDN` — starting MSISDN for generated UEs.
- `amf.hostname` — must match the AMF Service name (default `open5gs-amf`; changes if you
  use a different Helm release name for the open5gs chart).

## 4. Install with your edited files
```bash
helm install open5gs oci://registry-1.docker.io/gradiantcharts/open5gs --version 2.2.9 \
  -n open5gs --values ./5gSA-values.yaml
helm install ueransim-gnb oci://registry-1.docker.io/gradiant/ueransim-gnb --version 0.2.6 \
  -n open5gs --values ./gnb-ues-values.yaml
```
