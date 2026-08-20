% 5G SA Lab — Prerequisites and Learning Path
% The technologies and topics to know before using this project

# How to use this document

This lists every technology and topic the project touches, grouped by domain, with **why it
matters here** and **where it is used**. You don't need to master all of it before starting —
use the "Suggested learning order" at the end. Each domain also marks items as **[Must]**
(you'll be lost without it) or **[Helpful]** (deepens understanding / needed for the advanced parts).

\newpage

# Domain 1 — Networking fundamentals

The whole project is a network. This is the most important base.

| Topic | Why it matters here |
|-------|---------------------|
| IP addressing, subnets, CIDR [Must] | UE IPs (10.45.0.x / 192.168.100.x), container network (172.22.0.0/16), VM IP |
| Default gateway & routing [Must] | UE routes to the UPF gateway; downlink routing back to the gNB |
| NAT (Network Address Translation) [Must] | The UPF NATs UE traffic to the internet; VMware NAT gives the VM internet |
| TCP vs UDP, ports, sockets [Must] | GTP-U is UDP/2152; SBI is HTTP/2; understanding "port already in use" |
| ICMP / ping, traceroute [Must] | The end-to-end verification tool |
| DNS [Helpful] | Kubernetes service discovery resolves names like `open5gs-amf` |
| Tunneling / encapsulation [Must] | GTP-U wraps user packets between gNB and UPF |
| Network interfaces, TUN devices [Must] | `uesimtun0`, `ogstun` are virtual tunnel interfaces |
| SCTP [Helpful] | The transport for the N2 control link (NGAP over SCTP/38412) |
| Packet capture & analysis [Must] | tcpdump/Wireshark to inspect N2 and GTP-U |

# Domain 2 — Linux and the command line

Everything runs on Linux; you live in the terminal.

| Topic | Why it matters here |
|-------|---------------------|
| Bash shell: commands, pipes `\|`, redirects `>`, env vars [Must] | Every step; `curl … \| sh`, `export KUBECONFIG` |
| File system & paths, `cd`, `ls` [Must] | Running compose from the right folder |
| Permissions, `sudo`, ownership (`chown`) [Must] | Packet capture needs root; the kubeconfig fix |
| Processes: `ps`, `kill`, `killall` [Must] | Managing nr-gnb/nr-ue; clearing stale processes |
| Package management (`apt`) [Must] | Installing tshark, iproute2, build tools |
| `ip`, `ss` (iproute2) [Must] | Checking interfaces and which port is in use |
| systemd services [Helpful] | k3s and Docker run as services |
| Text editors (nano/vi) [Must] | Editing YAML/config files |

# Domain 3 — Virtualization

| Topic | Why it matters here |
|-------|---------------------|
| Virtual machines & hypervisors (VMware) [Must] | The lab runs in an Ubuntu VM |
| NAT vs bridged VM networking [Must] | NAT gives the VM internet; affects reachability |
| Guest/host, shared folders [Helpful] | Moving files between Windows and the VM |

# Domain 4 — Containers and Docker

| Topic | Why it matters here |
|-------|---------------------|
| Container concept (vs VM) [Must] | Open5GS NFs run as containers |
| Images & registries [Must] | Pulling images; the CNF idea |
| Dockerfile [Helpful] | The pcap exporter is built from one |
| Docker Compose: services, ports, volumes, env, networks [Must] | The whole `sa-deploy.yaml` core |
| Container networking & port publishing [Must] | The port-2152 conflict and its fix |

# Domain 5 — 5G / Telecom architecture (3GPP)

The core domain. This is what makes the project *5G* rather than generic containers.

| Topic | Why it matters here |
|-------|---------------------|
| RAN vs Core network [Must] | UERANSIM (RAN) vs Open5GS (Core) |
| 5G SA vs NSA [Must] | This lab is Standalone (SA) |
| Network Functions: AMF, SMF, UPF [Must] | The three you interact with most |
| Other NFs: NRF, AUSF, UDM, UDR, PCF, NSSF, SCP, BSF [Helpful] | The rest of the core; NF discovery via NRF |
| Interfaces: N1, N2, N3, N4, SBI [Must] | N2 (control), N3 (data), N4 (SMF-UPF) |
| Protocols: NGAP, NAS, GTP-U, PFCP, SCTP, HTTP/2 (SBI) [Must] | What flows on each interface; what the exporter decodes |
| gNB, UE, RRC [Must] | The tower, the phone, the radio signalling |
| Identifiers: PLMN (MCC/MNC), IMSI/SUPI, TAC, GUAMI [Must] | Configuring and matching the network identity |
| Network slicing: S-NSSAI (SST/SD), DNN/APN [Helpful] | Slice/data-network configuration |
| Authentication: K, OPc, SQN, 5G-AKA, SUCI [Must] | Why wrong keys fail; SQN resync; subscriber provisioning |
| PDU session, QoS / 5QI [Must] | The data connection the UE gets |
| Control plane vs user plane [Must] | The single most important operational concept in the lab |

# Domain 6 — Open5GS and UERANSIM (the specific tools)

| Topic | Why it matters here |
|-------|---------------------|
| Open5GS components & YAML configs [Must] | The core you deploy and tune |
| Open5GS WebUI / subscriber provisioning [Must] | Adding SIMs |
| UERANSIM gNB config (linkIp/ngapIp/gtpIp, amfConfigs) [Must] | The tower addressing that caused/fixed the packet loss |
| UERANSIM UE config (supi/key/op/opType, sessions) [Must] | The phone identity |

# Domain 7 — Observability (metrics, dashboards, alerts)

| Topic | Why it matters here |
|-------|---------------------|
| Metrics & time-series concept [Must] | Counters, rates |
| Prometheus: scraping, targets, exporters, PromQL [Must] | The pcap pipeline core; `rate()` for spikes |
| Grafana: data sources, dashboards, panels [Must] | Visualizing failures; the :9090 vs :9092 gotcha |
| Alertmanager: rules, routing, silences [Helpful] | Alerting on failure spikes |
| Log analysis & PCAP analysis [Must] | Diagnosing what failed |
| SLI/SLO thinking [Helpful] | Turning metrics into service objectives |

# Domain 8 — Kubernetes and Helm (cloud-native)

| Topic | Why it matters here |
|-------|---------------------|
| K8s core objects: Pod, Deployment, Service, Namespace [Must] | The CNF deployment building blocks |
| ConfigMap, Secret, PV/PVC, StatefulSet [Helpful] | Config injection and stateful MongoDB |
| kubectl (get/describe/logs/exec) [Must] | Operating and debugging the cluster |
| Cluster networking & service discovery (DNS, CNI) [Must] | Pods find each other by Service name |
| Helm: charts, values, releases, OCI repos [Must] | Deploying the 5G core as packages |
| k3s (lightweight Kubernetes) [Must] | The cluster you run |
| CNF concept [Must] | The whole point of the K8s phase |
| Multus CNI, SCTP on K8s, Operators/CRDs [Helpful] | The advanced, production-realistic extensions |

# Domain 9 — Programming, data formats, and tooling

| Topic | Why it matters here |
|-------|---------------------|
| YAML [Must] | Compose, Kubernetes manifests, Helm values, NF configs |
| JSON [Helpful] | The Grafana dashboard; API outputs |
| Bash scripting [Must] | helm / kubectl bring-up on the VM |
| Python basics + prometheus_client [Helpful] | The pcap exporter |
| Git / GitHub [Must] | Version control; publishing the project |
| tshark / Wireshark [Must] | Decoding 5G signalling |
| curl [Must] | Testing endpoints and the UE data path |

\newpage

# Suggested learning order

If you're starting fresh, learn in this order — each layer builds on the previous:

1. **Networking fundamentals** (IP, routing, NAT, TCP/UDP, ping, tunneling) — everything rests on this.
2. **Linux command line** (bash, permissions, processes, ip/ss, apt).
3. **Virtualization** (set up an Ubuntu VM with NAT).
4. **Containers & Docker Compose** (images, ports, the compose file model).
5. **5G architecture (3GPP)** — the core domain: RAN vs core, the NFs, N2/N3/N4, NGAP/NAS/GTP-U, identifiers, authentication, control vs user plane.
6. **Open5GS + UERANSIM** — apply domain 5 with these specific tools.
7. **Observability** (Prometheus, Grafana, Alertmanager, Wireshark).
8. **Kubernetes + Helm** — pods/deployments/services, kubectl, charts, k3s, CNFs.
9. **Programming glue** (YAML, bash, a little Python, Git).

# Minimum to get started vs. full mastery

- **Minimum to run the Docker lab:** Networking basics, Linux CLI, Docker Compose basics, and a working grasp of the 5G core/RAN split and the key identifiers (PLMN, IMSI, K/OPc). The guides fill the rest.
- **To run the Kubernetes phase:** add Kubernetes core objects, kubectl, and Helm basics.
- **To extend it (slicing, scale, external RAN, Multus, operators):** the [Helpful] items across domains 5 and 8, plus deeper Prometheus/Grafana and Python.

# Good study anchors (verify current versions/links yourself)

- 3GPP 5G system specs (TS 23.501 architecture, TS 23.502 procedures) — authoritative but dense.
- Open5GS documentation and UERANSIM wiki — the exact tools here.
- Kubernetes and Helm official docs; the CNCF ecosystem.
- Prometheus and Grafana official docs.

*(These are directions, not endorsements — check the current official sources.)*
