# 5G SA Core as CNFs on Kubernetes (Helm)

This deploys the same 5G Standalone network as the Docker Compose lab, but as
**Containerized Network Functions (CNFs)** on Kubernetes, packaged with **Helm** —
the way real 5G cores run in production. It uses the maintained **Gradiant** Helm
charts (`open5gs` + `ueransim-gnb`), which keep the Open5GS core you already know.

> Key difference from the Compose lab: here the RAN (UERANSIM) runs **inside** the
> cluster as a pod, so the gNB reaches the AMF by Kubernetes Service DNS — no host
> port juggling. (Attaching your *external*, host-native UERANSIM to an in-cluster
> core is a harder variant that needs SCTP NodePort/Multus; that's a later step.)

> **Verified working** on k3s v1.36: Open5GS chart `2.3.4`, `ueransim-gnb` (version
> auto-discovered), two UEs reaching `8.8.8.8` through `uesimtun0/1` at 0% packet loss.

---

## What changes vs Docker Compose

| Concept | Docker Compose | Kubernetes + Helm |
|---------|----------------|-------------------|
| Definition | one `sa-deploy.yaml` | Helm charts + values files |
| A service (amf, smf…) | a container | a **Deployment** + **Service** |
| MongoDB | a container + volume | a Pod + **PVC** |
| Config (`.env`) | env file | chart **values** → ConfigMaps |
| NF discovery | container IP | **Service DNS** (`open5gs-amf`, `open5gs-nrf`…) |
| Subscriber add | WebUI | `open5gs-dbctl` (populate) or WebUI via port-forward |
| Bring-up | `docker compose up -d` | `helm install` |
| Self-healing / rollout | none | built in (K8s restarts pods; `helm upgrade/rollback`) |

Resume line this unlocks: *"Deployed 5G SA core as containerized network functions (CNFs) on Kubernetes with Helm."*

---

## Step 0 — Prerequisites (install once, on the VM)

**What:** a local Kubernetes cluster, plus `kubectl` and `helm`.
**Why:** Helm needs a cluster to deploy into; `kubectl` is how you inspect it.

I recommend **k3s** (a lightweight, production-grade Kubernetes — one command, handles
SCTP and privileged pods well). Consider stopping the Compose core first to free RAM/ports:
```bash
cd ~/docker_open5gs && docker compose -f sa-deploy.yaml stop   # optional, frees resources
```

Install k3s and wire kubectl to it:
```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER ~/.kube/config
kubectl get nodes            # the node should show Ready
# make KUBECONFIG stick for future terminals (k3s kubectl otherwise reads a root-only file):
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
```

Install Helm:
```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## Step 1 — Create a namespace

**What:** a dedicated namespace `open5gs`.
**Why:** namespaces isolate this deployment's objects (pods, services, config) from anything else in the cluster — the standard way operators separate workloads.
```bash
kubectl create namespace open5gs
```

## Step 2 — Deploy the Open5GS 5G SA core

**What:** install the Gradiant `open5gs` chart with the SA values file.
**Why:** this creates every core NF (AMF, SMF, UPF, NRF, SCP, UDM/UDR/AUSF, PCF, MongoDB…) as Deployments/Services from one command. The values file: disables the 4G/EPC parts, enables a *populate* helper (`open5gs-dbctl`) that registers subscribers automatically, and disables the WebUI ingress.
```bash
# discover the current chart version (pinned tutorial versions go stale) then install
O5_VER=$(helm show chart oci://registry-1.docker.io/gradiantcharts/open5gs | awk '/^version:/{print $2}')
helm install open5gs oci://registry-1.docker.io/gradiantcharts/open5gs \
  --version "$O5_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/5gSA-values.yaml
```
Watch them come up:
```bash
kubectl -n open5gs get pods -w      # Ctrl+C when all are Running
```

## Step 3 — Deploy the UERANSIM RAN (gNB + UEs)

**What:** install the `ueransim-gnb` chart.
**Why:** this launches the simulated gNB *and* 2 UEs as pods, already configured to match the core's `mcc/mnc/sst/sd/tac`. The `amf.hostname` in its values points at the AMF's Service name (`open5gs-amf`) — that's the Kubernetes-DNS equivalent of the `amfConfigs.address` you set by hand in the Compose lab.
```bash
RAN_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-gnb | awk '/^version:/{print $2}')
helm install ueransim-gnb oci://registry-1.docker.io/gradiant/ueransim-gnb \
  --version "$RAN_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/gnb-ues-values.yaml
```

> Helm commands for Steps 1–3 are listed above. A private install wrapper is not published here.

## Step 4 — Verify the deployment

**What / why:** confirm each link of the chain, exactly like the Compose lab (N2, N4, N3, data).

Control plane — SMF↔UPF (N4) association:
```bash
kubectl -n open5gs logs deployment/open5gs-smf -f
```
Control plane — AMF accepts the gNB (N2 / NG Setup):
```bash
kubectl -n open5gs logs deployment/open5gs-amf -f
kubectl -n open5gs logs deployment/ueransim-gnb -f    # "NG Setup ... successful"
```
User plane — UEs got tunnels and can reach the internet:
```bash
kubectl -n open5gs exec deployment/ueransim-gnb-ues -ti -- bash
# inside the pod:
ip addr                       # expect uesimtun0, uesimtun1
ping -I uesimtun0 -c 4 8.8.8.8
# (a TLS curl may show 'certificate file' error 77 - cosmetic, the pod lacks CA certs;
#  use 'curl -k' or plain http to confirm application-layer connectivity)
exit
```
Optional — see the tunneled data at the UPF (needs `upf.containerSecurityContext.runAsUser: 0` in values):
```bash
kubectl -n open5gs exec deployment/open5gs-upf -ti -- bash -c "tcpdump -i ogstun"
```

## Step 5 — Manage subscribers

The `open5gs-populate` deployment already registered the initial subscriber(s). To add more:
```bash
kubectl -n open5gs exec deployment/open5gs-populate -ti -- \
  open5gs-dbctl add_ue_with_slice <imsi> <key> <opc> internet 1 <sd>
```
Or use the WebUI via a port-forward (then browse http://localhost:9999):
```bash
kubectl -n open5gs port-forward svc/open5gs-webui 9999:9999
```

## Step 6 — Align it with your lab (optional)

To make identifiers match your Compose lab (PLMN 999/70, your IMSI/K/OPc), download and edit
the values files first — see `values/customize-notes.md`. Then install with your local copies
instead of the URLs.

## Clean up
```bash
helm -n open5gs uninstall ueransim-gnb
helm -n open5gs uninstall open5gs
```

---

## How this maps to the "hard parts" of 5G-on-K8s

This first deployment deliberately keeps the RAN in-cluster to stay reliable. The advanced,
resume-worthy extensions from here:

- **SCTP + external RAN:** expose the AMF's N2 (SCTP) via NodePort/LoadBalancer so your
  *host-native* UERANSIM (the Compose-lab gNB) can attach to the in-cluster core.
- **Multus CNI:** give the NFs separate interfaces for N2/N3/N4 to mirror real deployments.
- **Operator model:** `Gradiant/open5gs-operator` manages Open5GS + subscribers via CRDs —
  the GitOps-style, declarative way to run it.

## Sources
- Gradiant 5G Charts — Open5GS + UERANSIM tutorial: https://gradiant.github.io/5g-charts/open5gs-ueransim-gnb.html
- Gradiant 5G Charts repo: https://github.com/Gradiant/5g-charts
- Open5GS Kubernetes reference (raw manifests, Multus): https://github.com/niloysh/open5gs-k8s
- Orange towards5gs-helm (Free5GC-based alternative): https://github.com/Orange-OpenSource/towards5gs-helm
