# Automated Container Security Monitoring & Remediation

A production-style, end-to-end security pipeline for Kubernetes environments that combines **runtime threat detection**, **centralized log correlation**, and **automated incident response** using open-source tooling.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
- [Detection Example](#detection-example)
- [Automated Remediation](#automated-remediation)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Project Status](#project-status)
- [Key Objectives](#key-objectives)
- [Disclaimer](#disclaimer)

---

## Overview

This project demonstrates how to build a **SOC-style cloud security pipeline** from scratch using only open-source tools. The pipeline detects suspicious container and pod behavior at runtime (via eBPF), correlates events in a centralized SIEM, and automatically triggers remediation workflows — all without manual intervention.

**Core capabilities:**
- Real-time detection of container escape attempts, privilege escalation, and suspicious process execution
- Kubernetes-aware alerting enriched with pod name, namespace, and container metadata
- Policy-driven automated response: isolate, restart, scale down, or notify
- Structured forensic logging for post-incident analysis

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes (k3s)                       │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────┐  │
│  │  Pod A   │    │  Pod B   │    │  Falco (eBPF probe)  │  │
│  │(workload)│    │(workload)│    │  syscall monitoring  │  │
│  └──────────┘    └──────────┘    └──────────┬───────────┘  │
└────────────────────────────────────────────┼────────────────┘
                                             │ alert (JSON)
                                    ┌────────▼────────┐
                                    │    Filebeat      │
                                    │  log shipper     │
                                    └────────┬─────────┘
                                             │ forward
                                    ┌────────▼─────────┐
                                    │     Graylog       │
                                    │  SIEM / alerting  │
                                    └────────┬──────────┘
                                             │ webhook trigger
                                    ┌────────▼──────────┐
                                    │       n8n          │
                                    │  workflow engine   │
                                    └────────┬───────────┘
                                             │
                              ┌──────────────┼──────────────┐
                              ▼              ▼              ▼
                         Delete pod    Scale down      Notify SOC
                                        deployment    (Slack/Email)
```

**Flow:** Detection → Correlation → Automation → Remediation

---

## Components

### Falco
- Runtime security engine using the **modern eBPF** probe (no kernel module required)
- Monitors Linux syscalls at the kernel level with near-zero overhead
- Detects threats such as:
  - Interactive shell spawned inside a container
  - Privileged process or sensitive file access
  - Unexpected outbound network connections
  - Container escape attempts
- Outputs structured JSON alerts enriched with **Kubernetes pod and namespace metadata**

### Kubernetes (k3s)
- Lightweight, single-node Kubernetes cluster suitable for home lab or edge environments
- Runs test workloads (e.g. `busybox`, `nginx` pods) to simulate real attack scenarios
- Falco integrates directly with the container runtime (containerd) via the CRI

### Filebeat
- Lightweight log shipper deployed as a DaemonSet
- Tails Falco's JSON output from the node filesystem
- Ships structured alert data to Graylog over a secure TCP/Beats input

### Graylog
- Centralized log management and SIEM platform
- Parses and indexes Falco alerts
- Applies stream rules to filter **security-relevant events only** (e.g. `priority >= WARNING`)
- Triggers HTTP alerts to the n8n webhook on rule match

### n8n
- Open-source workflow automation engine
- Webhook node receives the Graylog alert payload
- Parses fields: `k8s.pod.name`, `k8s.ns.name`, `container.id`, `rule`, `priority`
- Executes conditional remediation logic based on rule type and severity

---

## Detection Example

Falco detects an interactive shell spawned inside a running pod:

```json
{
  "rule": "Terminal shell in container",
  "priority": "WARNING",
  "time": "2026-02-19T14:32:01.000Z",
  "output_fields": {
    "k8s.pod.name": "testpod",
    "k8s.ns.name": "default",
    "container.id": "a3f8c1d29e45",
    "proc.name": "sh",
    "proc.cmdline": "sh",
    "user.name": "root",
    "evt.type": "execve"
  }
}
```

This alert is shipped by Filebeat to Graylog, which matches a stream rule and fires a webhook to n8n for automated handling.

---

## Automated Remediation

n8n workflows implement response logic based on alert severity and rule type:

| Trigger | Action |
|---|---|
| `Terminal shell in container` (WARNING) | Delete the offending pod |
| `Privileged container started` (ERROR) | Scale deployment to 0 replicas |
| `Outbound network connection` (WARNING) | Apply a NetworkPolicy to isolate the pod |
| Any severity >= WARNING | Send Slack / email notification to SOC |
| Any alert | Write structured incident log for forensics |

Remediation actions are executed via `kubectl` commands or the Kubernetes API invoked from within the n8n workflow.

---

## Prerequisites

| Requirement | Version |
|---|---|
| Linux host | Ubuntu 24.04 (recommended) |
| Docker / containerd | 24.x+ |
| k3s | v1.29+ |
| Falco | 0.38+ (modern eBPF) |
| Filebeat | 8.x |
| Graylog | 5.x |
| n8n | 1.x |

> **Note:** Falco's modern eBPF probe requires kernel version **5.8+** and BTF (BPF Type Format) support. Verify with `uname -r` and `ls /sys/kernel/btf/vmlinux`.

---

## Getting Started

### 1. Install k3s

```bash
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### 2. Install Falco (modern eBPF)

```bash
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] \
  https://download.falco.org/packages/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/falcosecurity.list

sudo apt-get update && sudo apt-get install -y falco
sudo systemctl start falco-modern-bpf
```

### 3. Configure Falco JSON output

Edit `/etc/falco/falco.yaml`:

```yaml
json_output: true
log_level: info
file_output:
  enabled: true
  filename: /var/log/falco/alerts.json
```

### 4. Deploy Filebeat

Configure Filebeat to tail `/var/log/falco/alerts.json` and forward to your Graylog Beats input (port `5044` by default).

### 5. Configure Graylog Stream & Alert

- Create a stream matching `source: falco`
- Add a stream rule for `priority` not equal to `DEBUG`
- Create an HTTP notification pointing to your n8n webhook URL

### 6. Import the n8n Workflow

- Open n8n at `http://<host>:5678`
- Import the workflow JSON from `n8n/remediation-workflow.json`
- Activate the workflow

### 7. Test the Pipeline

```bash
# Spin up a test pod
kubectl run testpod --image=busybox --restart=Never -- sleep 3600

# Trigger a detection
kubectl exec -it testpod -- sh
```

Falco should emit a `Terminal shell in container` alert within seconds, which propagates through the full pipeline.

---

## Project Status

| Component | Status |
|---|---|
| Falco detection pipeline | Complete |
| Kubernetes metadata enrichment | Complete |
| Filebeat log shipping | Complete |
| Graylog stream rules & alerting | Complete |
| n8n webhook integration | Complete |
| Policy-based remediation logic | In progress |
| NetworkPolicy isolation workflow | Planned |
| Multi-cluster support | Planned |

---

## Key Objectives

- Simulate a **real SOC-style cloud security pipeline** using production-grade open-source tools
- Demonstrate **detection-to-response automation** with sub-minute response time
- Show how **Red Team offensive knowledge** directly informs and improves defensive detection logic
- Build a **modular, extensible** architecture that can be adapted to production environments

---

## Disclaimer

This project is built for **educational and research purposes only**.  
Do not deploy automated remediation workflows in a production environment without proper change management, safeguards, dry-run modes, and human-in-the-loop approval gates.

---

## Author

**Aymen Boukadida**
