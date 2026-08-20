% 5G Lab — Kubernetes Command Practice Workbook
% Every kubectl/helm command: where to run it, what it does, and why

# How to use this workbook

Work the sessions in order. Each command is a step that tells you **where** (which terminal),
**what** it does, **why**, the **expected** result, and **if it fails** what to do. Everything
runs on the **Ubuntu VM**.

## Terminals — you mostly need only ONE

Unlike the Docker lab (where the gNB and UE had to stay running in separate terminals),
Kubernetes runs everything as **background pods**. So one terminal is enough for almost
everything. Only two commands "hold" a terminal until you press Ctrl+C:

| Terminal | Used for |
|----------|----------|
| **T1** | Everything: `kubectl`, `helm`, one-off commands |
| **T2** (optional) | Only for commands that keep running: `kubectl get pods -w` (live watch) and `kubectl port-forward` |

## Golden rules (the Kubernetes ones)

- **Set `KUBECONFIG` in every terminal.** New terminals get it from `~/.bashrc`; in an
  older terminal run `export KUBECONFIG=~/.kube/config` first, or you'll see "permission denied".
- **Always target the namespace:** put `-n open5gs` on every `kubectl`/`helm` command.
- **Don't pin stale chart versions.** Discover the current one with `helm show chart …`.
- **Early restarts are normal.** `pcf`/`udr` may restart a few times while waiting for the NRF.
- **Diagnose in one order:** `get` → `describe` → `logs`.
- **Deleting a pod doesn't remove the app.** Kubernetes recreates it (self-healing). To remove
  for real, use `helm uninstall` or `kubectl delete deployment`.

\newpage

# Session 1 — Set up the cluster (one time)

## Step 1.1 — Free resources  [T1]
```bash
cd ~/docker_open5gs && docker compose -f sa-deploy.yaml stop
```
*What/why:* stops the Compose core so k3s and the in-cluster core don't fight for RAM. Optional but recommended.

## Step 1.2 — Install k3s (Kubernetes)  [T1]
```bash
curl -sfL https://get.k3s.io | sh -
```
*What/why:* downloads and starts k3s as a background service — the cluster everything runs in. **Expected:** ends with "Starting k3s".

## Step 1.3 — Let kubectl reach the cluster  [T1]
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
kubectl get nodes
```
*What/why:* k3s writes a root-only config; these copy it to your home, make you the owner, and point kubectl at it. The `echo … >> ~/.bashrc` makes it automatic for future terminals. **Expected:** one node `Ready`. **If "permission denied":** you skipped/lost the `export` — run it again.

## Step 1.4 — Install Helm  [T1]
```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```
*What/why:* installs Helm (the Kubernetes package manager) and confirms it. **Expected:** a `v3.x` version.

\newpage

# Session 2 — Deploy the 5G core + radio

## Step 2.1 — Create the namespace  [T1]
```bash
kubectl create namespace open5gs
```
*Why:* an isolated compartment for all 5G objects. You'll target it with `-n open5gs`.

## Step 2.2 — Deploy the core  [T1]
```bash
O5_VER=$(helm show chart oci://registry-1.docker.io/gradiantcharts/open5gs | awk '/^version:/{print $2}')
helm install open5gs oci://registry-1.docker.io/gradiantcharts/open5gs \
  --version "$O5_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/5gSA-values.yaml
```
*What/why:* install Open5GS as CNFs. **Expected:** release `open5gs` deployed, then wait for pods.

## Step 2.3 — Deploy the RAN  [T1]
```bash
RAN_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-gnb | awk '/^version:/{print $2}')
helm install ueransim-gnb oci://registry-1.docker.io/gradiant/ueransim-gnb \
  --version "$RAN_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/gnb-ues-values.yaml
```
*Piece by piece:* the first line finds the current chart version (avoids the stale-version "not found" error); `helm install` names the release; `-n open5gs` picks the namespace; `--values <url>` applies the 5G-SA settings.

Then the RAN:
```bash
RAN_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-gnb | awk '/^version:/{print $2}')
helm install ueransim-gnb oci://registry-1.docker.io/gradiant/ueransim-gnb \
  --version "$RAN_VER" -n open5gs \
  --values https://gradiant.github.io/5g-charts/docs/open5gs-ueransim-gnb/gnb-ues-values.yaml
```

## Step 2.4 — Watch the pods start  [T2 or T1]
```bash
kubectl -n open5gs get pods -w
```
*What/why:* `-w` = watch (live updates). Ctrl+C when all are `Running`. `pcf`/`udr` restarting a few times is normal. **If you'd rather not block a terminal:** drop `-w` and run `kubectl -n open5gs get pods` a few times instead.

\newpage

# Session 3 — Inspect and verify

## Step 3.1 — See the pods and services  [T1]
```bash
kubectl -n open5gs get pods
kubectl -n open5gs get svc
```
*Why:* `get pods` shows health; `get svc` shows the Service names NFs use to find each other (e.g. `open5gs-amf`, `open5gs-nrf`).

## Step 3.2 — Confirm the tower connected  [T1]
```bash
kubectl -n open5gs logs deployment/ueransim-gnb | grep -i "NG Setup"
```
*What/why:* reads the tower's log; you want "NG Setup procedure is successful".

## Step 3.3 — Enter a UE and test the internet  [T1]
```bash
kubectl -n open5gs exec -ti deployment/ueransim-gnb-ues -- /bin/bash
```
*What/why:* opens a shell **inside** the UE pod (`-t` terminal, `-i` interactive). Then, inside:
```bash
ip addr | grep uesimtun
ping -I uesimtun0 -c 4 8.8.8.8
exit
```
*Why:* confirms the tunnels exist and a phone reaches the internet through the in-cluster core. **Expected:** 0% packet loss. (A TLS `curl` error 77 is cosmetic — the pod lacks CA certs.)

## Step 3.4 — When something's wrong: describe + logs  [T1]
```bash
kubectl -n open5gs describe pod <pod-name>       # events explain WHY a pod is stuck
kubectl -n open5gs logs deployment/<name>        # what the app printed
```

\newpage

# Session 4 — Subscribers and the WebUI

## Step 4.1 — Add a subscriber  [T1]
```bash
kubectl -n open5gs exec -ti deployment/open5gs-populate -- \
  open5gs-dbctl add_ue_with_slice <imsi> <key> <opc> internet 1 <sd>
```
*Why:* the core only admits SIMs it knows; `open5gs-dbctl` writes one into the database.

## Step 4.2 — Open the WebUI  [T2]
```bash
kubectl -n open5gs port-forward svc/open5gs-webui 9999:9999
```
*What/why:* forwards your local port 9999 to the WebUI pod. Leave it running (it blocks the
terminal — hence T2), then browse **http://localhost:9999**. Ctrl+C to stop.

\newpage

# Session 5 — Extend the network

## Step 5.1 — Add more UEs  [T1]
```bash
UES_VER=$(helm show chart oci://registry-1.docker.io/gradiant/ueransim-ues | awk '/^version:/{print $2}')
helm install ueransim-ues oci://registry-1.docker.io/gradiant/ueransim-ues \
  --version "$UES_VER" -n open5gs --set gnb.hostname=ueransim-gnb
```
*Why:* a dedicated chart adds phones on the existing tower; `--set gnb.hostname=ueransim-gnb`
tells them which tower to use. **Remember:** each new UE needs a matching subscriber (Session 4).

## Step 5.2 — See what a chart can be tuned to  [T1]
```bash
helm show values oci://registry-1.docker.io/gradiant/ueransim-ues --version "$UES_VER" | less
```
*Why:* prints every setting (e.g. UE count, starting IMSI/MSISDN) you can change with `--set`.

## Step 5.3 — Scale a workload  [T1]
```bash
kubectl -n open5gs scale deployment/<name> --replicas=2
```
*Why:* runs more copies. **Caution:** the core NFs (AMF/UPF/SMF/MongoDB) are **not** safely
duplicated this way — practice scaling the RAN/UE workloads, not the core.

\newpage

# Session 6 — Break things and watch Kubernetes heal

## Step 6.1 — Delete a pod and watch it come back  [T1]
```bash
kubectl -n open5gs delete pod <any-amf-pod-name>
kubectl -n open5gs get pods -w
```
*Why:* Kubernetes' **self-healing** — the Deployment immediately creates a replacement pod.
This is the headline difference from Docker Compose (which would just leave it dead). Ctrl+C when the new pod is `Running`.

## Step 6.2 — Wrong subscriber → auth failure  [T1]
Add a UE whose IMSI isn't registered (or a mismatched key), then read the gNB/AMF logs:
```bash
kubectl -n open5gs logs deployment/open5gs-amf | grep -i "auth\|reject"
```
*Why:* shows the core rejecting an unknown/invalid SIM — the same security behaviour as the Docker lab, observed via logs.

## Step 6.3 — Recover
```bash
kubectl -n open5gs rollout restart deployment/<name>   # cleanly restart an NF
```

\newpage

# Session 7 — Clean up

```bash
helm -n open5gs uninstall ueransim-gnb
helm -n open5gs uninstall open5gs
kubectl delete namespace open5gs
# remove Kubernetes entirely (optional):
/usr/local/bin/k3s-uninstall.sh
```
*Why:* `helm uninstall` removes a release; deleting the namespace clears anything left; the
k3s uninstall script removes the cluster itself.

\newpage

# Situational guide — "based on what I want to do"

| I want to... | Do this |
|--------------|---------|
| Set up the cluster | Session 1 |
| Deploy the 5G core + RAN | Session 2 (`helm install` Open5GS then UERANSIM) |
| Check health / verify a UE | Session 3 |
| Add subscribers / open the WebUI | Session 4 |
| Add more phones | Session 5.1 |
| See Kubernetes self-heal | Session 6.1 |
| Tear it all down | Session 7 |

# Situational guide — "based on a problem I see"

| Symptom | Cause | Fix |
|---------|-------|-----|
| `permission denied ... k3s.yaml` | `KUBECONFIG` not set in this terminal | `export KUBECONFIG=~/.kube/config` |
| `helm install ... not found` | Pinned chart version was removed | Discover it: `helm show chart <oci-url> \| awk '/^version:/{print $2}'` |
| Pod `Pending` | Node out of CPU/RAM | `kubectl -n open5gs describe pod <name>`; free resources |
| Pod `ImagePullBackOff` | Can't pull image (network/name) | `describe pod` to see the reason; check internet |
| Pod `CrashLoopBackOff` | App inside is erroring | `kubectl -n open5gs logs deployment/<name>` |
| `pcf`/`udr` restart a few times | Waiting for NRF | Normal — they settle to `Running` |
| curl in UE pod: error 77 | Pod lacks CA certificates | Cosmetic; use `curl -k` or `http://` |
| UE has no `uesimtun` / ping fails | Tower not attached, or subscriber missing | Check gNB log for "NG Setup"; check subscriber; `amf.hostname` = `open5gs-amf` |
| `helm install` name in use | Release already exists | `helm -n open5gs uninstall <name>` first |
| A pod I deleted keeps coming back | That's self-healing | To remove for real: `helm uninstall` or `kubectl delete deployment/<name>` |

\newpage

# kubectl / helm cheat sheet (all on the VM, T1)

| Command | What it does |
|---------|--------------|
| `kubectl -n open5gs get pods` | List pods + health |
| `kubectl -n open5gs get svc` | List services (the NF names) |
| `kubectl -n open5gs get all` | Everything in the namespace |
| `kubectl -n open5gs logs deployment/<name> -f` | Stream a pod's logs (`-f` follow) |
| `kubectl -n open5gs describe pod <name>` | Full details + events for one pod |
| `kubectl -n open5gs exec -ti deployment/<name> -- bash` | Shell inside a pod |
| `kubectl -n open5gs rollout restart deployment/<name>` | Cleanly restart an NF |
| `kubectl -n open5gs delete pod <name>` | Delete a pod (it self-heals) |
| `helm -n open5gs list` | List installed releases |
| `helm show chart <oci-url>` | Read a chart's metadata (find the version) |
| `helm show values <oci-url>` | Show every setting a chart accepts |
| `helm -n open5gs uninstall <name>` | Remove a release |

## Where do I run everything?
All commands run on the **Ubuntu VM** in **one terminal (T1)**. Use a **second terminal (T2)**
only for the two commands that keep running: `kubectl get pods -w` and `kubectl port-forward`.
