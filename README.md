# Kubernetes Workload Profiling with Calico and Pyroscope
This repository contains the configuration files and setup instructions for implementing Continuous Application Profiling in a Kubernetes environment using Calico Open Source and Grafana Pyroscope.

Continuous profiling allows you to identify performance bottlenecks and resource inefficiencies at a granular level (CPU, memory, etc.) in production environments with minimal overhead.

## Prerequisites

* Helm installed.
* Multipass installed (for local cluster setup).
* A Kubernetes cluster (setup instructions included below).
* Calico v3.28 or above

## 1. Cluster Setup (Local)
Use the following commands to launch a 3-node Kubernetes cluster using Multipass and K3s.

```bash
multipass launch -n c1-control -c 2 -m 8192M -d 50G 22.04 --cloud-init release/control-init.yaml
multipass launch -n c1-node1 -c 2 -m 4096M -d 50G 22.04 --cloud-init release/node-init.yaml
multipass launch -n c1-node2 -c 2 -m 4096M -d 50G 22.04 --cloud-init release/node-init.yaml
```

```bash
multipass transfer c1-control:/etc/rancher/k3s/k3s.yaml ./c1.config
```

```bash
export KUBECONFIG=$(pwd)/c1.config
CONTROL=$(multipass list --format csv | egrep c1-control | cut -d, -f3)
sed -i c1.config "s/127.0.0.1/$CONTROL/" 
```

## 2. Install Tigera operator

First install Tigera opreator
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
```

Use the following installation manifest to install Calico
```bash
kubectl create -f -<<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  registry: quay.io/
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - blockSize: 26
      cidr: 172.16.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
EOF
```

## 2. Enable Native Profiling in Calico

Calico 3.28+ supports native Go pprof profiling. By default, the profiling server is disabled.

### Enable the Profiling Server

Use the following command to enable the debug port (9096) on the default Felix configuration:
```bash
kubectl patch felixconfiguration default --type=merge -p='{"spec":{"debugPort":9096}}'
```

### Allow Remote Access (Optional)

To allow the profiling server to respond to requests from any origin (required for scraping by external agents):
```bash
kubectl patch felixconfiguration default --type=merge -p='{"spec":{"debugHost":"0.0.0.0"}}'
```

### Install Grafana Pyroscope
Pyroscope is used to collect and visualize profiling data.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install pyroscope grafana/pyroscope -n calico-monitoring
```

## 4. Scraping Profile Data (Native)
We use the Grafana Agent to scrape profiling data from our workloads.

### Method A: Native Profiling (Pull Method)
Scrape Calico’s native pprof metrics using the provided configuration:

```bash
helm install -n calico-monitoring pyroscope-agent grafana/grafana-agent \
  -f https://raw.githubusercontent.com/frozenprocess/pyroscope_observability_blog/main/scrape-pprof.yaml
```

### Method B: eBPF-based Profiling
For applications that do not have native profiling libraries, use eBPF probes. This method is transparent and requires no code changes.

```bash
helm upgrade -n calico-monitoring pyroscope-agent grafana/grafana-agent \
  -f https://raw.githubusercontent.com/frozenprocess/pyroscope_observability_blog/main/scrape-combine.yaml
```

### Note: eBPF profiling requires the agent to run in privileged mode.
