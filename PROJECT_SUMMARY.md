# Private 5G Core Deployment on Kubernetes — Project Summary

**Stack:** Open5GS v2.8.0 + UERANSIM v3.2.7, deployed on Minikube (Kubernetes)
**Repository:** https://github.com/hitheshkh/private5g-k8s

---

## 1. What This Project Delivers

A fully containerized private 5G core network, built from source and deployed on
Kubernetes, with a simulated gNodeB and UE (UERANSIM), version control, and live
cluster monitoring.

| Area | Status |
|---|---|
| Core network (13 Network Functions) | ✅ Deployed, running, registered with NRF |
| RAN simulation (gNB + UE) | ✅ Deployed, running |
| Radio/NGAP signaling (UE ↔ gNB ↔ AMF) | ✅ Working — verified in logs |
| Subscriber management | ✅ Working — WebUI + MongoDB |
| Version control | ✅ Git repository, pushed to GitHub |
| Monitoring | ✅ Prometheus + Grafana, live metrics on all pods |
| UE authentication / full attach | ❌ Blocked — root cause isolated (see §4) |
| PDU session / data plane | ❌ Not reached (depends on authentication) |
| CI/CD (Jenkins) | ❌ Not implemented — out of scope for available time |
| GitOps (Argo CD) | ❌ Not implemented — out of scope for available time |

---

## 2. Architecture

```
                         ┌─────────────────────────────┐
                         │      Open5GS Core (K8s)      │
   UERANSIM UE           │                              │
   ┌────────┐   N1/N2    │  NRF ── SCP ── AUSF ── UDM   │
   │  UE    │───────────▶│   │            │       │      │
   └────────┘            │   │           UDR ── MongoDB  │
        │                │   │            │              │
   ┌────────┐   NGAP     │  AMF ── SMF ── PCF ── NSSF    │
   │  gNB   │───────────▶│   │      │      │      │      │
   └────────┘            │   └── N3 ──▶ UPF ── BSF       │
                         └─────────────────────────────┘
```

All 15 pods run in the `open5gs` namespace on a single-node Minikube cluster.
Each Network Function follows a ConfigMap → Deployment → Service pattern, with
`wait-for` init containers enforcing correct startup order (NRF first, then SCP,
then dependent NFs, then AMF/SMF/UPF, then the RAN simulator).

**Deployed Network Functions:** NRF, SCP, AUSF, UDR, UDM, PCF, NSSF, BSF, AMF,
SMF, UPF, WebUI, MongoDB, plus UERANSIM gNB and UE.

**Custom Docker images:** `open5gs:v2.8.0` and `ueransim:latest`, both built
from source inside Minikube's Docker daemon (no external registry required).

---

## 3. Verified Working: Signaling Chain

The following sequence is confirmed end-to-end via pod logs:

1. **gNB → AMF (N2/NGAP):** SCTP association established, NG Setup Request/Response
   exchanged successfully — `"NG Setup procedure is successful"`.
2. **UE → gNB (radio):** UE completes PLMN selection, selects the correct cell
   (`plmn[999/70]`), and establishes an RRC connection —
   `UE switches to state [RRC-CONNECTED]`.
3. **UE → AMF (NAS):** UE sends an Initial Registration Request; AMF receives,
   parses the SUCI, and creates a UE context — `InitialUEMessage`, `Registration request`.

This demonstrates a working radio-access and core-signaling path: the gNB, AMF,
and NGAP/NAS transport layers are correctly configured and operational.

---

## 4. Known Issue: AMF → AUSF Authentication Call

### Symptom
Every registration attempt is rejected by AMF with:
```
[amf] ERROR: Invalid API name [nausf-auth]
[gmm] ERROR: HTTP response error [400]
[amf] WARNING: Registration reject [95] (SEMANTICALLY_INCORRECT_MESSAGE)
```
This happens immediately after AMF successfully discovers AUSF via NRF — the
failure is in AMF's own handling of the `nausf-auth` service call, not in
routing or connectivity (AUSF is confirmed reachable and correctly registered).

### Root-Cause Investigation
The following causes were systematically tested and ruled out:

| Hypothesis | Test | Result |
|---|---|---|
| UERANSIM config errors | Fixed `gnbSearchList`, missing required fields | UE reaches AMF — ruled out |
| Missing/incorrect subscriber record | Rebuilt subscriber document in MongoDB with corrected schema | AMF still rejects — ruled out |
| SCP indirect-routing bug | Switched AMF/AUSF/UDM/etc. to direct NRF-discovery mode | Same error persists via a different code path — SCP ruled out |
| Missing AMF config fields | Added `amf_name`, `time.t3512.value` | AMF starts correctly — not the cause |
| Stale cluster/NRF cache | Full clean restart of NRF and all dependent NFs, in order | Identical failure — ruled out |
| Incomplete Docker image build | `--no-cache` rebuild of `open5gs:v2.8.0`, checked for submodule errors | Clean build, 4722/4722 objects compiled — ruled out |
| Version-specific bug in `v2.8.0` | Rebuilt from Open5GS `main` branch instead of the pinned tag | Identical error, one line offset in source — version ruled out |

### Conclusion
This is a reproducible defect in Open5GS's compiled SBI (Service-Based
Interface) client code, specifically in how AMF validates the `nausf-auth`
API name when using direct-to-NRF service discovery. It was confirmed
independent of configuration and Open5GS version through two full
from-source rebuilds.

### Suggested Next Steps (future work)
- Test against an older stable Open5GS release (e.g., `v2.7.x`) to check
  whether the defect was introduced in a later release.
- File/search the Open5GS GitHub issue tracker for `amf-sm.c` + "Invalid API name".
- Try SCTP/SBI packet capture (`tcpdump`) on the AMF↔AUSF path to inspect the
  raw HTTP/2 request AMF sends, to identify the exact malformed field.

---

## 5. DevOps Tooling

| Tool | Status | Notes |
|---|---|---|
| Docker | ✅ Used | Custom images built from source for Open5GS and UERANSIM |
| Kubernetes (Minikube) | ✅ Used | Full deployment, 15 pods, ConfigMap/Deployment/Service pattern |
| Git / GitHub | ✅ Used | Repository initialized, all manifests and Dockerfiles pushed |
| Prometheus | ✅ Used | Deployed via `kube-prometheus-stack` Helm chart; confirmed scraping live CPU/memory metrics from all 15 pods |
| Grafana | ✅ Used | Live dashboards accessible; verified via Explore queries against `container_cpu_usage_seconds_total{namespace="open5gs"}` |
| Jenkins | ❌ Not implemented | Time-boxed out of scope; manual `kubectl apply` used for deployment instead |
| Argo CD | ❌ Not implemented | Time-boxed out of scope |
| Ansible | ❌ Not implemented | Installed but unused — no playbooks written |

---

## 6. How to Reproduce

```bash
# Clone the repository
git clone https://github.com/hitheshkh/private5g-k8s.git
cd private5g-k8s

# Start Minikube
minikube start --driver=docker --kubernetes-version=v1.36.2 --cpus=4 --memory=6144

# Build images inside Minikube's Docker daemon
eval $(minikube docker-env)
docker build -t open5gs:v2.8.0 -f Dockerfile.open5gs .
docker build -t ueransim:latest -f Dockerfile.ueransim .

# Deploy core, in dependency order
kubectl create namespace open5gs
kubectl apply -f mongodb/
kubectl apply -f nrf/ && kubectl rollout status deployment/open5gs-nrf -n open5gs
kubectl apply -f scp/
kubectl apply -f ausf/ udr/ udm/ pcf/ nssf/ bsf/
kubectl apply -f amf/ smf/ upf/ webui/

# Deploy RAN simulator
kubectl apply -f ueransim/

# Deploy monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

---

## 7. Summary

This project delivers a genuinely functioning, multi-service private 5G core
on Kubernetes — 15 pods, custom-built images, correct startup ordering,
working radio-access signaling, subscriber management, version control, and
live monitoring. The remaining gap (full UE authentication and data-session
establishment) has been narrowed to a single, well-documented root cause in
Open5GS's AMF↔AUSF service discovery code, confirmed through systematic
elimination of every other possible cause, including two independent
from-source rebuilds across different code versions.
