% 5G SA Lab — Kubernetes Workflow (Complete, Step by Step)
% Deploy the 5G core + radio as CNFs with Helm, in order, with the files to use

# How to use this workflow

This is the complete Kubernetes-based path from an empty machine to a 5G core running as
containerized network functions (CNFs). Do the phases in order. Everything runs on the
**Ubuntu VM**, in **one terminal** (Kubernetes runs pods in the background — no foreground
processes to babysit).

## Files this workflow uses

| File / folder | Role |
|---------------|------|
| `kubernetes/README.md` | Step-by-step Helm commands |
| `kubernetes/values/customize-notes.md` | How to align PLMN/IMSI with your lab |

\newpage

# Phase 0 — Prerequisites (one time)

**Do:** free resources, install k3s, wire up kubectl, install Helm.
```bash
cd ~/docker_open5gs && docker compose -f sa-deploy.yaml stop     # free RAM (optional)
curl -sfL https://get.k3s.io | sh -                              # install Kubernetes (k3s)
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER ~/.kube/config
export KUBECONFIG=~/.kube/config                                 # the permission-denied fix
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc         # for future terminals
kubectl get nodes                                                # expect: node Ready
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```
**Guide:** `docs/5G_Kubernetes_Guide.docx` Section 2 (every command explained).
**End with:** a `Ready` node and Helm installed.

# Phase 1 — Create the namespace

```bash
kubectl create namespace open5gs
```
**End with:** the `open5gs` compartment for all objects.

# Phase 2 — Deploy the core (CNFs)

```bash
O5_VER=$(helm show chart oci://registry-1.docker.io/gradiantcharts/open5gs | awk '/^version:/{print $2}')
helm install open5gs oci://registry-1.docker.io/gradiantcharts/open5gs \
  --version "$O5_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/5gSA-values.yaml
kubectl -n open5gs get pods        # wait until all Running (pcf/udr may restart a few times - normal)
```
**End with:** every NF pod `1/1 Running`.

# Phase 3 — Deploy the radio (gNB + UEs)

```bash
RAN_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-gnb | awk '/^version:/{print $2}')
helm install ueransim-gnb oci://registry-1.docker.io/gradiant/ueransim-gnb \
  --version "$RAN_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/gnb-ues-values.yaml
kubectl -n open5gs get pods        # ueransim-gnb + ueransim-gnb-ues Running
```
**End with:** the tower and 2 UEs running as pods.

# Phase 4 — Verify

```bash
kubectl -n open5gs logs deployment/ueransim-gnb | grep -i "NG Setup"     # tower connected
kubectl -n open5gs exec -ti deployment/ueransim-gnb-ues -- /bin/bash
#   inside the pod:
ip addr | grep uesimtun
ping -I uesimtun0 -c 4 8.8.8.8
exit
```
**End with:** UE ping to `8.8.8.8` at 0% loss (a TLS `curl` error 77 is cosmetic — missing CA certs).

# Phase 5 — Subscribers and WebUI

```bash
kubectl -n open5gs exec -ti deployment/open5gs-populate -- \
  open5gs-dbctl add_ue_with_slice <imsi> <key> <opc> internet 1 <sd>
kubectl -n open5gs port-forward svc/open5gs-webui 9999:9999    # then browse http://localhost:9999
```

# Phase 6 — Extend (more UEs / pods)

```bash
# more phones on the same tower:
UES_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-ues | awk '/^version:/{print $2}')
helm install ueransim-ues oci://registry-1.docker.io/gradiant/ueransim-ues \
  --version "$UES_VER" -n open5gs --set gnb.hostname=ueransim-gnb
# each new UE needs a matching subscriber (Phase 5)
```
**Guide:** `docs/5G_Kubernetes_Guide.docx` Section 7 (also covers `helm show values` and scaling caveats).

# Phase 7 — Troubleshoot

```bash
kubectl -n open5gs get pods                 # which pod is unhealthy?
kubectl -n open5gs describe pod <name>      # why (events at the bottom)?
kubectl -n open5gs logs deployment/<name>   # what is the app saying?
```
**Guide:** `docs/5G_Kubernetes_Guide.docx` Section 8 (symptom -> fix table: KUBECONFIG, stale version, Pending/ImagePullBackOff/CrashLoopBackOff, curl 77, name-in-use).

# Phase 8 — Clean up

```bash
helm -n open5gs uninstall ueransim-gnb
helm -n open5gs uninstall open5gs
kubectl delete namespace open5gs
# remove Kubernetes entirely (optional):  /usr/local/bin/k3s-uninstall.sh
```

\newpage

# Whole Kubernetes path as one command sequence

```bash
# prereqs (once)
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER ~/.kube/config
export KUBECONFIG=~/.kube/config
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# deploy core + RAN (see kubernetes/README.md)
kubectl create namespace open5gs
O5_VER=$(helm show chart oci://registry-1.docker.io/gradiantcharts/open5gs | awk '/^version:/{print $2}')
helm install open5gs oci://registry-1.docker.io/gradiantcharts/open5gs \
  --version "$O5_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/5gSA-values.yaml
RAN_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-gnb | awk '/^version:/{print $2}')
helm install ueransim-gnb oci://registry-1.docker.io/gradiant/ueransim-gnb \
  --version "$RAN_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/gnb-ues-values.yaml
# verify
kubectl -n open5gs get pods
kubectl -n open5gs exec -ti deployment/ueransim-gnb-ues -- ping -I uesimtun0 8.8.8.8
```
