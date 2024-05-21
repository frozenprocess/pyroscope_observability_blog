requirements
Helm
multipass

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


```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
```

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



```bash
kubectl patch felixconfiguration default --type=merge -p='{"spec":{"debugPort":9096}}'
```

```bash
kubectl patch felixconfiguration default --type=merge -p='{"spec":{"debugHost":"0.0.0.0"}}'
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install pyroscope grafana/pyroscope -n calico-monitoring
helm install -n calico-monitoring pyroscope-agent grafana/grafana-agent -f scrape-pprof.yaml
```