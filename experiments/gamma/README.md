---
title: Talos & Cilium
date: 2026/11/20
date-modified: 2026/11/21
abstract: |
  This experiment deploys a 3-node Talos Linux cluster (control plane plus two workers) in Docker and runs Cilium*as the only CNI and service load balancer.
---

# Deployment

![](timeline.svg)

Set environment variable before deployment:

```bash
talos_version=$(talosctl version --client | awk '/Tag:/ {print $2; exit}')
cilium_version="$(cilium version --client | sed -n 's/^cilium image (stable): v\([0-9][0-9.]*\)$/\1/p')"
export TALOSCONFIG="$PWD/.talos/talosconfig"
export KUBECONFIG="$PWD/.talos/kubeconfig"
```

Create the cluster, and set the endpoint[^OerD2]:

[^OerD2]: Deploy Cilium CNI, Siderlabs Documentation  
<https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium>

```bash
talosctl cluster create docker \
  --name gamma --subnet "10.5.1.0/24" --workers 2 \
  --state "$PWD/.talos/clusters" \
  --talosconfig-destination "$TALOSCONFIG" \
  --image "ghcr.io/siderolabs/talos:$talos_version" \
  --config-patch-controlplanes=@talos-network.yaml
# Without no CNI, the kubelet reports NotReady

# configure the control plane endpoint
talosctl config node 10.5.1.2

# (later) tear down this experiment
talosctl cluster destroy --name gamma --state "$PWD/.talos/clusters"
rm -rf "$PWD/.talos"
```

Install the Cilium CNI plugin 

```bash
cilium install --version "$cilium_version" --values cilium.yaml
# wait until Cilium is installed
cilium status --wait
```

# Configuration

## Talos

Patches deletes Flannel CNI plugin and disables `kube-proxy` (both replaced by Cilium)

```yaml
# talos-network.yaml
apiVersion: v1alpha1
kind: KubeFlannelCNIConfig
$patch: delete
---
apiVersion: v1alpha1
kind: KubeProxyConfig
enabled: false
```
- `KubeFlannelCNIConfig` — removes the default Flannel CNI
- `patch: delete` — JSON6902 marker; Talos deletes this resource
- `KubeProxyConfig` with `enabled: false` — disables `kube-proxy`

There should be no information about Flannel (CNI):

```bash
talosctl get machineconfig v1alpha1 -o jsonpath='{.spec}' \
      | yq -N '.kind // "(v1alpha1)"' | grep -c Flannel
```

**`kube-proxy` is disabled**

```bash
talosctl get machineconfig v1alpha1 -o jsonpath='{.spec}' \
      | yq 'select(.kind == "KubeProxyConfig")'
```
```txt
apiVersion: v1alpha1
kind: KubeProxyConfig
enabled: false # Enable or disable kube-proxy deployment on cluster bootstrap.
image: registry.k8s.io/kube-proxy:v1.37.1 # The container image used in the kube-proxy manifest.
# Provide configuration for the kube-proxy.
config: {}
```

**KubePrism configuration**

```bash
talosctl -n 10.5.1.3 get kubeprismstatuses
```
```txt
NODE       NAMESPACE   TYPE              ID                 VERSION   HOST             HEALTHY
10.5.1.3   k8s         KubePrismStatus   k8s-loadbalancer   3         127.0.0.1:7445   true
```

* KubePrism is its per-node local proxy to the API servers.
  - One KubePrism pod runs on every node
  - Listens on `localhost:7445` (`k8sServiceHost` / `k8sServicePort` in `cilium.yaml`)
  - Balances each request across the control plane API servers
  - Local tools (kubelet, kubectl) reach the API via loopback
* Why with Cilium?
  - Cilium runs in host network mode. It cannot use `kubernetes.default.svc` to reach the API server.
  - Without KubePrism it uses the external cluster endpoint.
  - If that endpoint fails, Cilium breaks: pods stop scheduling, Service IPs stop working.
  - KubePrism gives Cilium a local, always-available endpoint at `localhost:7445`.

## Cilium

**Cilium (CNI) works as service load balancer**

- Assigns pod IPs from `node.spec.podCIDR` (IPAM mode kubernetes)
- Implements ClusterIP, NodePort, ExternalIPs directly in eBPF
- No iptables rules: service routing lives in eBPF maps
- Pod identity and network policy come from the same eBPF data

`cilium-cli` does not auto-detect Talos and requires following configuration:

```yaml
ipam:
  mode: kubernetes
kubeProxyReplacement: true
k8sServiceHost: localhost
k8sServicePort: 7445
cgroup:
  autoMount:
    enabled: false
    hostRoot: /sys/fs/cgroup
securityContext:
  capabilities:
    ciliumAgent: [CHOWN, KILL, NET_ADMIN, NET_RAW, IPC_LOCK, SYS_ADMIN, SYS_RESOURCE, DAC_OVERRIDE, FOWNER, SETGID, SETUID]
    cleanCiliumState: [NET_ADMIN, SYS_ADMIN, SYS_RESOURCE]
```

**Why these settings in `cilium.yaml`**

- `kubeProxyReplacement: true` pairs with `KubeProxyConfig.enabled: false`
- `cgroup.autoMount` off: Talos mounts cgroup v2 itself
- `SYS_MODULE` dropped: Talos blocks pod kernel module loading
- `ipam.mode: kubernetes` reuses Talos's `KubeNetworkConfig` pod CIDR

Examine the network configuration:

```bash
kubectl -n kube-system exec ds/cilium \
      -- cilium-dbg status | grep -E 'KubeProxyReplacement|Routing|Kubernetes:'
```
```txt
Kubernetes:              Ok         1.37 (v1.37.1) [linux/amd64]
KubeProxyReplacement:    True   [eth0  10.5.1.4 (Direct Routing)]
Routing:                 Network: Tunnel [vxlan]   Host: Legacy
```

# Experiments

## Kube Proxy

This section is a test that Cilium handles Services after `kube-proxy` is disabled.

```bash
kubectl create deployment web --image=nginx:1.29-alpine --replicas=2
kubectl expose deployment web --port=80
kubectl get service web
```
```txt
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.110.65.239   <none>        80/TCP    92s
```

- Creates a 2-replica nginx Deployment.
- Exposes it as a `ClusterIP` Service (`web`, port 80).

```bash
kubectl -n kube-system exec ds/cilium \
      -- cilium-dbg service list \
      | grep -A2 "$(kubectl get svc web -o jsonpath='{.spec.clusterIP}')"
```
```txt
6    10.110.65.239:80/TCP    ClusterIP      1 => 10.244.1.133:80/TCP (active)     
                                            2 => 10.244.2.62:80/TCP (active) 
```

- `cilium-dbg service list` shows the Service IP mapped to both Pod IPs.
- This proves Cilium does the endpoint selection and load balancing.

```bash
talosctl -n 10.5.1.3 get nftableschains  # should be empty
```

`talosctl get nftableschains` is empty. This proves `kube-proxy` is not writing iptables/nftables rules.

```bash
# Deletes the Service and Deployment
kubectl delete service web; kubectl delete deployment web
```

**Confirms Cilium's `kubeProxyReplacement` fully replaces `kube-proxy` for Service traffic**

## KubePrism

Gamma runs Cilium with `kubeProxyReplacement`; KubePrism is the matching local API access path.

Restarts the `cilium` daemon set to trigger a fresh connection.

```bash
kubectl -n kube-system rollout restart ds/cilium
kubectl -n kube-system rollout status ds/cilium
```
```txt
daemonset.apps/cilium restarted
Waiting for daemon set "cilium" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "cilium" rollout to finish: 2 of 3 updated pods are available...
daemon set "cilium" successfully rolled out
```

**Prove that Cilium reaches the API through KubePrism**

```bash
kubectl -n kube-system logs ds/cilium | grep -m3 -i 'localhost:7445'
```
```txt
Found 3 pods, using pod/cilium-m7sb4
time=2026-11-21T04:35:42.417922798Z level=info msg="Establishing connection to apiserver" module=agent.infra.k8s-client ipAddr=https://localhost:7445
```


## Capabilities

Show the capabilities of the `cilium-agent` container:

```bash
kubectl -n kube-system get ds cilium \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="cilium-agent")].securityContext.capabilities.add}{"\n"}'
```
```txt
["CHOWN","KILL","NET_ADMIN","NET_RAW","IPC_LOCK","SYS_ADMIN","SYS_RESOURCE","DAC_OVERRIDE","FOWNER","SETGID","SETUID"]
```