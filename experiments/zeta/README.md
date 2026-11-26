---
title: Talos Maintenance & Recovery 
date: 2026/11/26
abstract: |
  This experiment runs a three-control-plane Talos Linux QEMU cluster with a Layer 2 VIP for the Kubernetes API and verifies that the API stays reachable during VIP failover and a hung node, then performs etcd maintenance, replaces a control plane node, and recovers the whole cluster from an etcd snapshot.
---

# Deployment 

```bash
# Set the environment
talos_version=$(talosctl version --client | awk '/Tag:/ {print $2; exit}')
export TALOSCONFIG="$PWD/.talos/talosconfig"
export KUBECONFIG="$PWD/.talos/kubeconfig"

# Start the cluster
sudo --preserve-env=HOME,TALOSCONFIG,KUBECONFIG \
      "$(command -v talosctl)" cluster create qemu \
            --name zeta --cidr "10.5.1.0/24" \
            --state "$PWD/.talos/clusters" \
            --talosconfig-destination "$TALOSCONFIG" \
            --talos-version "$talos_version" \
            --config-patch-controlplanes=@patches/controlplane/vip.yaml \
            --controlplanes 3 --workers 1
```

If firewalld is active on the host, either stop or...

```bash
# ...add the cluster bridge to the trusted zone
# Read the cluster bridge name from the configuration
bridge=$(grep '^bridgename:' "$PWD/.talos/clusters/zeta/state.yaml" | awk '{print $2}')
# Add it to the trusted zone
sudo firewall-cmd --zone=trusted --add-interface="$bridge"
sudo firewall-cmd --get-active-zones
```



Later, tear down of the cluster

```bash
sudo --preserve-env=HOME \
      "$(command -v talosctl)" cluster destroy \
            --name zeta --state "$PWD/.talos/clusters"
sudo rm -rf "$PWD/.talos/"
```

# Configuration

Adjust permissions for the configuration files

```bash
sudo chown "$(id -u):$(id -g)" "$TALOSCONFIG" "$KUBECONFIG"
# Keep the provisioner files
sudo mv *.yaml .talos/
sudo chown -R "$(id -u):$(id -g)"  .talos/*.yaml
```

Set endpoints in the Talos client configuration:

```bash
talosctl config endpoint 10.5.1.2 10.5.1.3 10.5.1.4
talosctl config info
```
```txt
Current context:     zeta
Nodes:               not defined
Endpoints:           10.5.1.2, 10.5.1.3, 10.5.1.4
Roles:               os:admin
Certificate expires: 1 year from now (2027-11-26)
```

## Kubernetes API

API server runs on all three control plane nodes (clients should not depend on a node IP) ...three ways clients to reach the API:

1. **External load balancer**
   - Load balancer in front of the three API servers.
   - Cluster endpoint in the config
   - Here the QEMU provisioner runs it on the host, at the gateway IP.
   - It sends each request to one healthy API server.
   - Data center uses a real hardware or software load-balancers
2. **KubePrism**
   - A small TCP load balancer runs on each node, on loopback only.
   - It reads the API server list from the discovery data.
   - Local clients use it: the kubelet, Cilium, host-network pods.
   - It is not for clients outside the cluster.
4. **Layer 2 VIP**
   - No extra component. The nodes share one virtual IP (VIP).
   - `etcd` elects one owner node at a time.
   - The owner adds 10.5.1.50 to its link and sends gratuitous ARP.
   - Traffic to the VIP hits the current owner.
   - All control plane nodes must be on one flat (layer 2) network.

Patch to the control plane nodes:

```yaml
# patches/controlplane/vip.yaml
apiVersion: v1alpha1
kind: Layer2VIPConfig
name: 10.5.1.50
link: net0
---
apiVersion: v1alpha1
kind: KubeAPIServerConfig
certExtraSANs:
  - 10.5.1.50
```

After the patch, the API server gets a new certificate that includes the VIP.

* **`Layer2VIPConfig`**
  - `Talos setting for a Layer 2 virtual IP.
  - `name: 10.5.1.50` = the VIP address itself.
  - `link: net0` = the interface to add it on.
  - `net0` is the QEMU alias for the virtio NIC.
  - Effect: marks the node able to own the VIP.
* **`KubeAPIServerConfig`**
  - Add the VIP to the API server serving certificate.
  - Why: TLS checks the certificate against the address in the URL.
  - Talos does not add the VIP by itself.

## Layer 2 VIP

Why the VIP is not the Talos endpoint?

* The VIP only exists while `etcd` is healthy:
  - The VIP is created and moved by the `etcd` election.
  - If `etcd` fails, the VIP disappears: there is no owner.
* `talosctl` must be able to reach the node to repair `etcd`...
* ..so its endpoint cannot depend on `etcd`.

**`talosctl` points at the node IPs, not the VIP.**

Two different endpoints:

- `kubectl` talks to the Kubernetes API (port 6443). It can use the VIP.
- `talosctl` talks to the Talos API (port 50000). It manages the OS itself.
- The VIP is only useful for the Kubernetes-API, not a Talos-management.

# Experiments

## Using a VIP

Check if `etcd` daemon's are running, preparing, or failed:

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 service etcd
```

Ask control plane nodes about `etcd` members:

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 etcd status
```
```txt
NODE       MEMBER             DB SIZE   IN USE             LEADER             RAFT INDEX   RAFT TERM   RAFT APPLIED INDEX   LEARNER   PROTOCOL   STORAGE   ERRORS
10.5.1.2   b4af3900cf70cd64   2.4 MB    2.4 MB (100.00%)   b4af3900cf70cd64   2032         2           2032                 false     3.7.3      3.7.0     
10.5.1.3   3c1f719b69067cf8   2.3 MB    2.3 MB (100.00%)   b4af3900cf70cd64   2032         2           2032                 false     3.7.3      3.7.0     
10.5.1.4   a77e5c41ed890d16   2.3 MB    2.3 MB (100.00%)   b4af3900cf70cd64   2032         2           2032                 false     3.7.3      3.7.0   
```

Find the control plane node holding the VIP:

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 get addresses \
      | grep -E 'NODE|10.5.1.50'
```
```txt
NODE       NAMESPACE   TYPE            ID                   VERSION   ADDRESS       LINK
10.5.1.2   network     AddressStatus   enp0s6/10.5.1.50/32  1         10.5.1.50/32  enp0s6
```

Use the VIP to access the Kubernetes API:

```bash
kubectl --server "https://10.5.1.50:6443" get nodes
```
```txt
NAME                  STATUS   ROLES           AGE   VERSION
zeta-controlplane-1   Ready    control-plane   23m   v1.37.1
zeta-controlplane-2   Ready    control-plane   23m   v1.37.1
zeta-controlplane-3   Ready    control-plane   23m   v1.37.1
zeta-worker-1         Ready    <none>          23m   v1.37.1
```

```bash
openssl s_client -connect "10.5.1.50:6443" </dev/null 2>/dev/null \
      | openssl x509 -noout -ext subjectAltName
```
```txt
X509v3 Subject Alternative Name: 
    DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, DNS:localhost, DNS:zeta-controlplane-1, IP Address:10.5.1.1, IP Address:10.5.1.2, IP Address:10.5.1.50, IP Address:10.96.0.1, IP Address:127.0.0.1
```

## Kubeprism

Check health of KubePrism TCP load balancer:

```bash
talosctl -n 10.5.1.5 get kubeprismstatuses
```
```txt
NODE       NAMESPACE   TYPE              ID                 VERSION   HOST             HEALTHY
10.5.1.5   k8s         KubePrismStatus   k8s-loadbalancer   2         127.0.0.1:7445   true
```

List of API server addresses (backends) that the local KubePrism load balancer 

```bash
talosctl -n 10.5.1.5 get kubeprismendpoints -o yaml | yq .spec
```
```txt
endpoints:
  - host: 10.5.1.1
    port: 6443
  - host: 10.5.1.2
    port: 6443
  - host: 10.5.1.50
    port: 6443
  - host: 10.5.1.3
    port: 6443
  - host: 10.5.1.4
    port: 6443
```

## VIP Failover

Monitor the connection to the Kubernetes API...

```bash
# ..in a dedicated terminal (Ctrl-C to stop)
while true; do
  echo -n "$(date +%T) "
  kubectl --server "https://10.5.1.50:6443" \
        --request-timeout 2s get --raw /readyz
  echo
  sleep 1
done
```

Reboot the node holding the VIP:

```bash
talosctl -n 10.5.1.2 reboot
```

Connections continue to work while the VIP moves to another node:

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 get addresses \
      | grep -E 'NODE|10.5.1.50'
```
```txt
NODE       NAMESPACE   TYPE            ID                    VERSION   ADDRESS      LINK
10.5.1.4   network     AddressStatus   enp0s6/10.5.1.50/32   1         10.5.1.50/32 enp0s6
```

## Node Hang

Keep the monitoring loop from the previous experiment running

```bash
# Stop the VM holding the VIP
echo stop | sudo socat - UNIX-CONNECT:build/state/zeta/zeta-controlplane-3.monitor
```

Connection to the Kubernetes API will break temporarily with following errors: 

```txt
11:12:03 Unable to connect to the server: dial tcp 10.5.1.50:6443: connect: no route to host
11:12:04 Unable to connect to the server: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

State of the `etcd` with a failing node: 

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 etcd status
```
```txt
NODE       MEMBER             DB SIZE   IN USE            LEADER             RAFT INDEX   RAFT TERM   RAFT APPLIED INDEX   LEARNER   PROTOCOL   STORAGE   ERRORS
10.5.1.2   b4af3900cf70cd64   3.1 MB    1.5 MB (50.20%)   3c1f719b69067cf8   13535        4           13535                false     3.7.3      3.7.0
10.5.1.3   3c1f719b69067cf8   3.1 MB    1.5 MB (49.67%)   3c1f719b69067cf8   13535        4           13535                false     3.7.3      3.7.0
error from node 10.5.1.4: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.5.1.4:50000: connect: no route to host"
```

The VIP jumps again to one for the remaining control plane nodes:

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 get addresses \
      | grep -E 'NODE|10.5.1.50'
```
```txt
NODE       NAMESPACE   TYPE            ID                   VERSION   ADDRESS       LINK
10.5.1.3   network     AddressStatus   enp0s6/10.5.1.50/32  1         10.5.1.50/32  enp0s6
```

```bash
# Continue the VM
echo cont | sudo socat - UNIX-CONNECT:.talos/clusters/zeta/zeta-controlplane-3.monitor
```

Time diverged during the stop of the VM (NTP poll interval up to 2048s)

```bash
talosctl -e 10.5.1.2 -n 10.5.1.4 time
```
```txt
NODE       NTP-SERVER            NODE-TIME                                 NTP-SERVER-TIME
10.5.1.4   time.cloudflare.com   2026-11-26 10:11:48.045174704 +0000 UTC   2026-11-26 10:16:52.113790626 +0000 UTC
```

`etcd` recovers:

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 etcd status
```
```txt
NODE       MEMBER             DB SIZE   IN USE            LEADER             RAFT INDEX   RAFT TERM   RAFT APPLIED INDEX   LEARNER   PROTOCOL   STORAGE   ERRORS
10.5.1.3   3c1f719b69067cf8   3.1 MB    2.4 MB (79.52%)   3c1f719b69067cf8   15455        4           15455                false     3.7.3      3.7.0     
10.5.1.2   b4af3900cf70cd64   3.1 MB    2.4 MB (79.63%)   3c1f719b69067cf8   15455        4           15455                false     3.7.3      3.7.0     
10.5.1.4   a77e5c41ed890d16   3.0 MB    2.4 MB (77.96%)   3c1f719b69067cf8   15455        4           15455                false     3.7.3      3.7.0     
```

## Etcd Maintenance

```bash
talosctl -n 10.5.1.2 etcd snapshot etcd.snapshot
```
```txt
etcd snapshot saved to "etcd.snapshot" (3076128 bytes)
snapshot info: hash 140f00b8, revision 13359, total keys 565, total size 3076096
```

Move the leadership. Use the node that is the leader now:

```bash
talosctl -n 10.5.1.3 etcd forfeit-leadership
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 etcd status
```
```txt
NODE       MEMBER             DB SIZE   IN USE            LEADER             RAFT INDEX   RAFT TERM   RAFT APPLIED INDEX   LEARNER   PROTOCOL   STORAGE   ERRORS
10.5.1.2   b4af3900cf70cd64   3.1 MB    1.6 MB (53.00%)   a77e5c41ed890d16   20068        5           20068                false     3.7.3      3.7.0     
10.5.1.4   a77e5c41ed890d16   3.0 MB    1.6 MB (53.23%)   a77e5c41ed890d16   20068        5           20068                false     3.7.3      3.7.0     
10.5.1.3   3c1f719b69067cf8   3.1 MB    1.6 MB (53.01%)   a77e5c41ed890d16   20068        5           20068                false     3.7.3      3.7.0     
```

Reclaims the unused (fragmented) space in one `etcd` member's database:

```bash
talosctl -n 10.5.1.2 etcd defrag
talosctl -n 10.5.1.2 etcd status
talosctl -n 10.5.1.2 etcd alarm list
```

## Control Plane Replacement

Reads the active machine configuration and saves just its `.spec` (the config body))

```bash
talosctl -n 10.5.1.2 get mc v1alpha1 -o jsonpath='{.spec}' > nodes/controlplane.yaml
```

Gracefully removes a node from the cluster...

* ...or a control plane node it first removes its etcd member
* ...node comes back as a fresh machine in maintenance mode

```bash
talosctl -n 10.5.1.3 reset --reboot
# list the etcd members 
talosctl -n 10.5.1.2 etcd members
```
```txt
NODE       ID                 HOSTNAME              PEER URLS               CLIENT URLS             LEARNER
10.5.1.2   a77e5c41ed890d16   zeta-controlplane-3   https://10.5.1.4:2380   https://10.5.1.4:2379   false
10.5.1.2   b4af3900cf70cd64   zeta-controlplane-1   https://10.5.1.2:2380   https://10.5.1.2:2379   false
```

Sends the control plane machine config to the (reset) node...

```bash
talosctl -n 10.5.1.3 apply-config --insecure -f nodes/controlplane.yaml
```
```txt
Applied configuration without a reboot
```

...the node reboots into the cluster as a new member

```bash
talosctl -n 10.5.1.2 etcd members 
```
```txt
NODE       ID                 HOSTNAME              PEER URLS               CLIENT URLS             LEARNER
10.5.1.2   8e4e319c0e6c1499   zeta-controlplane-2   https://10.5.1.3:2380   https://10.5.1.3:2379   false
10.5.1.2   a77e5c41ed890d16   zeta-controlplane-3   https://10.5.1.4:2380   https://10.5.1.4:2379   false
10.5.1.2   b4af3900cf70cd64   zeta-controlplane-1   https://10.5.1.2:2380   https://10.5.1.2:2379   false
```

## Snapshot Recovery

Modify the Etcd configuration before and after a snapshot:

```bash
kubectl create configmap before-snapshot --from-literal=note=kept
talosctl -n 10.5.1.2 etcd snapshot etcd.snapshot
kubectl create configmap after-snapshot --from-literal=note=lost
```
```txt
configmap/before-snapshot created
etcd snapshot saved to "etcd.snapshot" (2949152 bytes)
snapshot info: hash 9dffcadb, revision 24003, total keys 420, total size 2949120
configmap/after-snapshot created
```

Destroy all Etcd members at the same time


```bash
for node in 10.5.1.2 10.5.1.3 10.5.1.4
do
  talosctl -e "$node" -n "$node" reset \
        --graceful=false --reboot \
        --system-labels-to-wipe=EPHEMERAL --wait=false
done
```

Wait until Etcd is in the `Preparing` state

- `bootstrap --recover-from` needs an **empty** etcd data directory.
- If the node still has data (etcd running with the old data), it fails.
- What `Preparing` means:
  - `etcd` is up, has no data, and is not in a cluster.
  - It waits for a bootstrap: recover from a snapshot, or join an existing cluster.
  - It stays there until you bootstrap. Until then the API and the VIP are down.

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 service etcd
```
```txt
NODE     10.5.1.3
ID       etcd
STATE    Preparing
HEALTH   ?
#...
NODE     10.5.1.4
ID       etcd
STATE    Preparing
HEALTH   ?
#...
NODE     10.5.1.2
ID       etcd
STATE    Preparing
HEALTH   ?
#...
```

Recover one node. The other two join the recovered cluster on their own.

```bash
talosctl -n 10.5.1.2 bootstrap --recover-from=etcd.snapshot
```
```txt
recovering from snapshot "etcd.snapshot": hash 9dffcadb, revision 24003, total keys 420, total size 2949120
```

Verify the configuration before the snapshot is present:

```bash
kubectl get configmap before-snapshot after-snapshot
```
```txt
NAME              DATA   AGE
before-snapshot   1      11m
Error from server (NotFound): configmaps "after-snapshot" not found
```