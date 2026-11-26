---
title: Talos Upgrade & Rollback
date: 2026/11/23
date-modified: 2026/11/25
---

# Deployment

Local Platform Qemu, Sidero Documentation  
<https://docs.siderolabs.com/talos/v1.14/platform-specific-installations/local-platforms/qemu>

Creates QEMU cluster: 1 controlplane, 1 worker, network 10.5.1.0/24

```bash
# Set the environment
talos_version=v1.14.5
schematic_id=ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
export TALOSCONFIG="$PWD/.talos/talosconfig"
export KUBECONFIG="$PWD/.talos/kubeconfig"

# Start the cluster
sudo --preserve-env=HOME,TALOSCONFIG,KUBECONFIG \
      "$(command -v talosctl)" cluster create qemu \
            --name eps --cidr "10.5.1.0/24" --workers 1 \
            --state "$PWD/.talos/clusters" \
            --talosconfig-destination "$TALOSCONFIG" \
            --talos-version "$talos_version" --schematic-id $schematic_id \
            --presets iso,maintenance
```



If firewalld is active on the host, add the cluster bridge to the trusted zone:

```bash
# Read the cluster bridge name from the configuration
bridge=$(grep '^bridgename:' "$PWD/.talos/clusters/eps/state.yaml" | awk '{print $2}')
# Add it to the trusted zone
sudo firewall-cmd --zone=trusted --add-interface="$bridge"
sudo firewall-cmd --get-active-zones
```

Adjust permissions for the configuration files

```bash
sudo chown "$(id -u):$(id -g)" "$TALOSCONFIG"
# Keep the provisioner files
sudo mv *.yaml .talos/
sudo chown -R "$(id -u):$(id -g)"  .talos/*.yaml
```

Later, tear down of the cluster

```bash
sudo --preserve-env=HOME \
      "$(command -v talosctl)" cluster destroy \
            --name eps --state "$PWD/.talos/clusters"
sudo rm -rf "$PWD/.talos/"
```

# Configuration

Node aconfiguration

```bash
# Generates a secrets bundle...
talosctl gen secrets -o secrets.yaml

# ...renders the machine configs...
talosctl gen config install https://10.5.1.2:6443 \
      --with-secrets secrets.yaml --with-docs=false --with-examples=false \
      --talos-version=v1.14.5 --kubernetes-version=1.37.0 \
      --install-image factory.talos.dev/metal-installer/${schematic_id}:v1.14.5 \
      --install-disk /dev/vda --output nodes/

# ...apply the configuration
talosctl apply-config --insecure --nodes 10.5.1.2 \
      --file nodes/controlplane.yaml \
      --config-patch @patches/controlplane-1.yaml

talosctl apply-config --insecure -n 10.5.1.3 \
      --file nodes/worker.yaml \
      --config-patch @patches/worker-1.yaml
```

This will take some time, logs are available in `.talos/`

```bash
export TALOSCONFIG="$PWD/nodes/talosconfig"
talosctl config endpoint 10.5.1.2
talosctl config node 10.5.1.2 10.5.1.3
talosctl config info
 ```

 ```bash
talosctl get unattendedinstallstatus -n 10.5.1.2,10.5.1.3
```


Bootstrap Kubernetes

```bash
talosctl bootstrap -n 10.5.1.2

# verify
talosctl service etcd
talosctl get staticpodstatus
```

# Experiments

```bash
talosctl get unattendedinstallstatus -n 10.5.1.2,10.5.1.3
```
```txt
WARNING: 10.5.1.2: server version 1.14.5 is older than client version 1.14.6, 10.5.1.3: server version 1.14.5 is older than client version 1.14.6
NODE       NAMESPACE   TYPE                      ID                   VERSION   PHASE
10.5.1.2   runtime     UnattendedInstallStatus   unattended-install   1         installed
10.5.1.3   runtime     UnattendedInstallStatus   unattended-install   1         installed
```

```bash
talosctl get securitystate -o yaml | yq .spec
```
```txt
secureBoot: false
selinuxState: enabled, permissive
bootedWithUKI: true
moduleSignatureEnforced: true
#...
```

Fields

- **`secureBoot: false`** — UEFI Secure Boot is not active on these VMs.
  - Your QEMU cluster does not use the `iso-secureboot` preset or SecureBoot images.
- **`selinuxState: enabled, permissive`** — SELinux runs in permissive mode.
  - It logs policy violations but does not block them.
  - This is the Talos default.
- **`bootedWithUKI: true`** — Node booted via a Unified Kernel Image.
  - One signed `.efi` file holds kernel, initrd, and kernel command line.
  - Replaces separate `vmlinuz` + `initramfs` files.
- **`moduleSignatureEnforced: true`** — Kernel accepts only signed modules.
  - Unsigned kernel modules cannot load.



## Upgrade Talos

Watch the upgrade from Kubernetes perspective:

```bash
export KUBECONFIG="$PWD/nodes/kubeconfig"
talosctl -n 10.5.1.2 kubeconfig "$KUBECONFIG"
kubectl get nodes --watch
```

Verify the current version of Talos Linux

```bash
talosctl -n 10.5.1.3 version
```
```txt
#...
Server:
        NODE:        10.5.1.3
        Tag:         v1.14.5
```

Upgrade both nodes with:

```bash
talos_version=v1.14.6
schematic_id=ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
image=factory.talos.dev/metal-installer/${schematic_id}:${talos_version}
talosctl -n 10.5.1.3 upgrade --image "$image"
```
```txt
10.5.1.3: pulled image factory.talos.dev/metal-installer/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515@sha256:4a777b6a2fcf956d20bf680dd33a32200722abe337f628fe5842fd1f8a096c88
10.5.1.3: upgrade completed
delta-worker-1: node drained
WARNING: 10.5.1.3: server version 1.14.5 is older than client version 1.14.6
watching nodes: [10.5.1.3]
    * 10.5.1.3: post check passed
delta-worker-1: node uncordoned
```

```bash
talosctl get bootedentries
```
```txt
NODE       NAMESPACE   TYPE          ID            VERSION   BOOTED ENTRY
10.5.1.2   runtime     BootedEntry   bootedentry   1         Talos-v1.14.6.efi
10.5.1.3   runtime     BootedEntry   bootedentry   1         Talos-v1.14.6.efi
```

## Upgrade Kubernetes


Show plan and pre-pull the images:

```bash
talosctl upgrade-k8s -n 10.5.1.2 --dry-run
```
```txt
automatically detected the lowest Kubernetes version 1.37.0
discovered controlplane nodes ["10.5.1.2"]
discovered worker nodes ["10.5.1.3"]
> "10.5.1.2": Talos version 1.14.6 is compatible with Kubernetes version 1.37.1
> "10.5.1.3": Talos version 1.14.6 is compatible with Kubernetes version 1.37.1
> "10.5.1.2": pre-pulling registry.k8s.io/kube-apiserver:v1.37.1
> "10.5.1.2": pre-pulling registry.k8s.io/kube-controller-manager:v1.37.1
> "10.5.1.2": pre-pulling registry.k8s.io/kube-scheduler:v1.37.1
> "10.5.1.2": pre-pulling ghcr.io/siderolabs/kubelet:v1.37.1
> "10.5.1.3": pre-pulling ghcr.io/siderolabs/kubelet:v1.37.1
#...
```

Run the upgrade on all nodes:

```bash
talosctl upgrade-k8s -n 10.5.1.2
```

Verify the upgrade:

```bash
kubectl get nodes
```
```txt
NAME                   STATUS   ROLES           AGE   VERSION
delta-controlplane-1   Ready    control-plane   33m   v1.37.1
delta-worker-1         Ready    <none>          33m   v1.37.1
```

## Rollback

```bash
talosctl -n 10.5.1.3 rollback
```

```bash
talosctl -n 10.5.1.3 version --short
```
```txt
Client:
Talos v1.14.6
Server:
        NODE:        10.5.1.3
        Tag:         v1.14.5
        SHA:         dad8d3ed
        Built:       
        Go version:  go1.26.10
        OS/Arch:     linux/amd64
        Enabled:     RBAC
```

## Rollout

```bash
talosctl -n 10.5.1.3 upgrade --image "$image" --no-reboot
```
```bash
talosctl -n 10.5.1.3 get bootedentries
```
```txt
WARNING: 10.5.1.3: server version 1.14.5 is older than client version 1.14.6
NODE       NAMESPACE   TYPE          ID            VERSION   BOOTED ENTRY
10.5.1.3   runtime     BootedEntry   bootedentry   1         Talos-v1.14.5.efi
```

```bash
talosctl -n 10.5.1.3 reboot
```
```txt
WARNING: 10.5.1.3: server version 1.14.5 is older than client version 1.14.6
watching nodes: [10.5.1.3]
    * 10.5.1.3: post check passed
```
```bash
talosctl -n 10.5.1.3 get bootedentries
```
```txt
NODE       NAMESPACE   TYPE          ID            VERSION   BOOTED ENTRY
10.5.1.3   runtime     BootedEntry   bootedentry   1         Talos-v1.14.6~1.efi
```