
# Deployment

Builds a full 3-node Talos Kubernetes cluster as Docker containers

```bash
talos_version=$(talosctl version --client | awk '/Tag:/ {print $2; exit}')
export TALOSCONFIG="$PWD/.talos/talosconfig"
export KUBECONFIG="$PWD/.talos/kubeconfig"
talosctl cluster create docker \
  --name beta --subnet "10.5.1.0/24" \
  --state "$PWD/.talos/clusters" \
  --talosconfig-destination "$TALOSCONFIG" \
  --image "ghcr.io/siderolabs/talos:$talos_version" \
  --workers 1

# configure the endpoint
talosctl config node 10.5.1.2 
```


```bash
talosctl cluster destroy --name beta --state "$PWD/.talos/clusters"
rm -rf "$PWD/.talos"
```

# Configuration

## Nodes

Shows the MachineConfig resource of nodes

```bash
talosctl -n 10.5.1.2,10.5.1.3 get machineconfig
```
```txt
NODE       NAMESPACE   TYPE            ID           VERSION
10.5.1.2   config      MachineConfig   persistent   1
10.5.1.2   config      MachineConfig   v1alpha1     1
10.5.1.3   config      MachineConfig   persistent   1
10.5.1.3   config      MachineConfig   v1alpha1     1
```

* MachineConfig resource holds the node configuration
* **Active vs Persistent**
  - `v1alpha1` runs in memory. The node works with it now.
  - `persistent` is saved on disk. It applies after reboot.
  - Editing the active config alone does not survive a restart.
* How Changes Work
  - `talosctl apply-machineconfig` changes the active `v1alpha1` copy.
  - Add `--persist` to also write the `persistent` copy.
  - A reboot loads the `persistent` config into the active one.

## Secrets

Following illustrates the process of creating a cluster configuration (done implicitly by `talos create cluster`)

```bash
# make a secret bundle
talosctl gen secrets -o secrets.yaml
```

* What are secrets used for? 
  - All nodes must share one secret bundle.
  - It holds PKI keys, API tokens, and client certificates.
  - Nodes use it to authenticate to the control plane.
* The Steps
  - `talosctl gen secrets -o secrets.yaml` makes the bundle.
  - `talosctl gen config` writes per-role machine configs.

**Create the cluster configuration**

```bash
talosctl gen config demo https://10.5.1.2:6443 --with-docs=false \
       --with-examples=false --with-secrets secrets.yaml -o demo/
```
```txt
generating PKI and tokens
Created demo/controlplane.yaml
Created demo/worker.yaml
Created demo/talosconfig
```
- The cluster name (`demo`) becomes the output directory.
- The endpoint `https://10.5.1.2:6443` sets the API address.
- `--with-docs=false` and `--with-examples=false` strip comments.
- `--with-secrets secrets.yaml` embeds that bundle in them.

List the resources of each file

```bash
yq -N '.kind // "(v1alpha1)"' demo/controlplane.yaml
yq -N '.kind // "(v1alpha1)"' demo/worker.yaml
```

**Validate a configuration**

```bash
talosctl validate --config demo/worker.yaml --mode container --strict
talosctl validate --config demo/worker.yaml --mode metal --strict
```
```txt
demo/worker.yaml is valid for container mode
demo/worker.yaml is valid for metal mode
```

* `--mode` is the platform: `container` (Docker), `metal` (bare metal & QEMU).
* Some settings are valid only on some platforms.

## Patches

Add a node label to a copy of the worker.yaml (cf. `examples/node-labels.yaml`):


```bash
talosctl machineconfig patch demo/worker.yaml \
      -p @examples/node-labels.yaml -o demo/worker-blue.yaml
# Compare the files
diff demo/worker.yaml demo/worker-blue.yaml
```
```txt
66a67,68
> labels:
>     lab.example.com/tier: blue
```

Read the configuration of a running node:

```bash
talosctl -n 10.5.1.3 get machineconfig v1alpha1 \
         -o jsonpath='{.spec}' > demo/worker-running.yaml
```

**Patch a running node**:

```bash
# Dry-run to sdee the expected changes
talosctl -n 10.5.1.3 patch mc --dry-run -p @examples/node-labels.yaml
```
```txt
#...
@@ -56,6 +56,8 @@
 apiVersion: v1alpha1
 kind: KubeNodeConfig
 nodeIP: {}
+labels:
+    lab.example.com/tier: blue
#...
```
```bash
# Apply the patch
talosctl -n 10.5.1.3 patch mc -p @examples/node-labels.yaml
```
```txt
patched MachineConfigs.config.talos.dev/v1alpha1 at the node 10.5.1.3
Applied configuration without a reboot
```

Verify the application of the patch:

```bash
kubectl get node beta-worker-1 -L lab.example.com/tier
```
```txt
NAME            STATUS   ROLES    AGE   VERSION   TIER
beta-worker-1   Ready    <none>   91m   v1.37.1   blue
```

## Try Mode

Try Mode tests a config change on a live node with **automatic rollback**.

```bash
talosctl -n 10.5.1.3 patch mc --mode try --timeout 45s -p @examples/static-host.yaml
```
```txt
patched MachineConfigs.config.talos.dev/v1alpha1 at the node 10.5.1.3
Applied configuration without a reboot
The config is applied in 'try' mode and will be automatically reverted back in 45s
```
```bash
# test immediately (after 45s the entry should be gone again) 
talosctl -n 10.5.1.3 read /etc/hosts
```
```txt
#...
10.5.1.200 registry.lab.example.com registry.lab.example.com
```

**The Workflow**
- Apply in try mode and watch the effect.
- If it works, re-apply the same patch without `--mode try`.
- That second apply makes the change permanent.

```bash
talosctl -n 10.5.1.3 patch mc -p @examples/static-host.yaml

# confirm that the resource persisted 
talosctl -n 10.5.1.3 get statichosts
```
```txt
patched MachineConfigs.config.talos.dev/v1alpha1 at the node 10.5.1.3
Applied configuration without a reboot
NODE       NAMESPACE   TYPE         ID                         VERSION   ADDRESSES
10.5.1.3   network     StaticHost   beta-worker-1              2         ["10.5.1.3","10.244.0.0"]
10.5.1.3   network     StaticHost   registry.lab.example.com   1         ["10.5.1.200"]
```

**Rules**
- Try mode never reboots the node.
- If the apply fails, it returns an error.
- It suits risky changes like network or API server tweaks.

# Experiments

## Add & Delete

**Create a file `/etc/demo.txt`** on one of the worker nodes:

```yaml
# examples/etc-file.yaml
apiVersion: v1alpha1
kind: EtcFileConfig
name: demo.txt
mode: 0o644
contents: |
  Configured by a machine config document
```

```bash
# Apply the patch...
talosctl -n 10.5.1.3 patch mc -p @examples/etc-file.yaml
# ...and verify the demo.txt exists
talosctl -n 10.5.1.3 read /etc/demo.txt
```
```txt
Configured by a machine config document
```

```bash
talosctl -n 10.5.1.3 get etcfilestatuses
```
```txt
NODE       NAMESPACE   TYPE            ID                              VERSION
10.5.1.3   files       EtcFileStatus   cri/conf.d/base-spec.json       2
10.5.1.3   files       EtcFileStatus   cri/conf.d/cri.toml             2
10.5.1.3   files       EtcFileStatus   demo.txt                        3
10.5.1.3   files       EtcFileStatus   hosts                           6
10.5.1.3   files       EtcFileStatus   machine-id                      2
10.5.1.3   files       EtcFileStatus   ssl/certs/ca-certificates.crt   2
```

**Delete the file `/etc/demo.txt`**

```yaml
# examples/etc-delete.yaml
apiVersion: v1alpha1
kind: EtcFileConfig
name: demo.txt
$patch: delete
```
```bash
talosctl -n 10.5.1.3 patch mc -p @examples/etc-delete.yaml
talosctl -n 10.5.1.3 read /etc/demo.txt
```
```txt
error reading: rpc error: code = NotFound desc = stat /etc/demo.txt: no such file or directory
```

## DNS over TLS

Patch an node to resolve DNS over TLS (DoT) to Quad9:

```yaml
# examples/dns-over-tls.yaml
apiVersion: v1alpha1
kind: ResolverConfig
nameservers:
  - address: 9.9.9.9               # DNS server IP
    protocol: DoT                  # Encrypt queries with DNS over TLS
    tlsServerName: dns.quad9.net   # TLS server name (SNI) for the cert
```

Modify the resolver via patch:

```bash
# Apply the patch (automatically revoked after 1 minute)
talosctl -n 10.5.1.3 patch mc --mode try -p @examples/dns-over-tls.yaml 
# verify the new resolver configuration
talosctl -n 10.5.1.3 get resolvers -o yaml
```
```txt
#...
spec:
    dnsServers:
        - 9.9.9.9
    nameServers:
        - addr: 9.9.9.9
          protocol: DoT
          tlsServerName: dns.quad9.net
#...
```

Lists the live upstream connections

```bash
talosctl -n 10.5.1.3 get dnsupstreams
```
```txt
NODE       NAMESPACE   TYPE          ID                 VERSION   HEALTHY   ADDRESS
10.5.1.3   network     DNSUpstream   #000 DoT 9.9.9.9   2         true      9.9.9.9:853
10.5.1.3   network     DNSUpstream   #001 DoT 9.9.9.9   1         true      9.9.9.9:853
```

## Invalid Patch

Patch a file locally and validate the output:

```bash
talosctl machineconfig patch demo/worker.yaml \
         -p @examples/invalid.yaml -o demo/worker-invalid.yaml
talosctl validate --config demo/worker-invalid.yaml --mode container
```
```txt
1 error occurred:
        * KubeNodeConfig: invalid node labels: 1 error occurred:
        * label value length exceeds limit of 63: "this-value-is-much-too-long-for-a-kubernetes-label-value-so-validation-fails"
```

Send the patch to the node:

```bash
talosctl -n 10.5.1.3 patch mc -p @examples/invalid.yaml
```
```txt
patched MachineConfigs.config.talos.dev/v1alpha1 at the node 10.5.1.3
rpc error: code = InvalidArgument desc = 1 error occurred:
        * KubeNodeConfig: invalid node labels: 1 error occurred:
        * label value length exceeds limit of 63: "this-value-is-much-too-long-for-a-kubernetes-label-value-so-validation-fails"
```

## Editor

Open the configuration of the worker in your editor:

```bash
talosctl -n 10.5.1.3 edit machineconfig --mode try
# change KubeNodeConfig tier to green
```

Verify the change (reverted after 1 minute):

```bash
kubectl get node beta-worker-1 -L lab.example.com/tier
```
```txt
NAME            STATUS   ROLES    AGE    VERSION   TIER
beta-worker-1   Ready    <none>   3h3m   v1.37.1   green
```