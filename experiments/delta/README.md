---
title: Talos with Qemu
date: 2026/11/21
date-modified: 2026/11/23
abstract: |
  This document builds a local Talos cluster (1 control plane, 1 worker) on QEMU, from ISO boot in maintenance mode through a manual install to a Ready cluster, then experiments with its partition layout, machine configs, and node reset flows.
---

# Deployment

Local Platform Qemu, Sidero Documentation  
<https://docs.siderolabs.com/talos/v1.14/platform-specific-installations/local-platforms/qemu>

Creates QEMU cluster: 1 controlplane, 1 worker, network 10.5.1.0/24

```bash
# Set the environment
talos_version=$(talosctl version --client | awk '/Tag:/ {print $2; exit}')
export TALOSCONFIG="$PWD/.talos/talosconfig"
export KUBECONFIG="$PWD/.talos/kubeconfig"

# Start the cluster
sudo --preserve-env=HOME,TALOSCONFIG,KUBECONFIG \
      "$(command -v talosctl)" cluster create qemu \
            --name delta --cidr "10.5.1.0/24" --workers 1 \
            --state "$PWD/.talos/clusters" \
            --talosconfig-destination "$TALOSCONFIG" \
            --talos-version "$talos_version" \
            --presets iso,maintenance

# Later tear down of the cluster
sudo --preserve-env=HOME \
      "$(command -v talosctl)" cluster destroy \
            --name delta --state "$PWD/.talos/clusters"
sudo rm -rf "$PWD/.talos/"
```
- **`--presets iso,maintenance`**
  -  VMs boot ISO from Image Factory, stay in maintenance mode
  -  Machine configs are written to the working dir.

If firewalld is active on the host, add the cluster bridge to the trusted zone:

```bash
# Read the cluster bridge name from the configuration
bridge=$(grep '^bridgename:' "$PWD/.talos/clusters/delta/state.yaml" | awk '{print $2}')
# Add it to the trusted zone
sudo firewall-cmd --zone=trusted --add-interface="$bridge"
sudo firewall-cmd --get-active-zones
```
```txt
#...
trusted
  interfaces: talos4f4a9410
  sources: 10.5.1.2/32 10.5.1.3/32
```

Summary of the cluster state 

```bash
talosctl cluster show \
      --provisioner qemu --name delta --state "$PWD/.talos/clusters/"
```
```txt
PROVISIONER           qemu
NAME                  delta
NETWORK NAME          delta
NETWORK CIDR          10.5.1.0/24
NETWORK GATEWAY       10.5.1.1
NETWORK MTU           1500
KUBERNETES ENDPOINT   https://10.5.1.1:6443

NODES:

NAME                   TYPE           IP         CPU    RAM      DISK
delta-controlplane-1   controlplane   10.5.1.2   2.00   2.1 GB   11 GB
delta-worker-1         worker         10.5.1.3   2.00   2.1 GB   11 GB
```

Adjust permissions for the configuration files

```bash
sudo chown "$(id -u):$(id -g)" "$TALOSCONFIG"
# Keep the provisioner files
sudo mv *.yaml .talos/
sudo chown -R "$(id -u):$(id -g)"  .talos/*.yaml
```

# Configuration

Verify that connection to a node works, `--insecure` uses the **maintenance API**:

```bash
talosctl -n 10.5.1.2 version --insecure
```
```txt
#...
Server:
        NODE:        10.5.1.2
        Tag:         v1.14.6
        SHA:         ede45c42
#...
```

Show the storage configuration of a node:

```bash
talosctl -n 10.5.1.2 get disks --insecure
```
```txt
NODE       NAMESPACE   TYPE   ID      VERSION   SIZE     READ ONLY   TRANSPORT   ROTATIONAL   WWID   MODEL          SERIAL
10.5.1.2   runtime     Disk   loop0   2         4.1 kB   true                                                       
10.5.1.2   runtime     Disk   loop1   2         94 MB    true                                                       
10.5.1.2   runtime     Disk   sr0     2         365 MB   true        sata        true                QEMU DVD-ROM   
10.5.1.2   runtime     Disk   sr1     2         0 B      true        sata                            QEMU DVD-ROM   
10.5.1.2   runtime     Disk   vda     2         11 GB    false       virtio      true     
```

* `sr0` is the CD-ROM drive with the ISO. 
* `loop1` is the Talos root file system (SquashFS) in RAM.
* `vda` is empty. This is the install disk.

## Secrets & Machines

Prepare the install on the nodes local storage by generating the machine configuration:

```bash
# Generates a secrets bundle
talosctl gen secrets -o secrets.yaml
# Renders the machine configs
talosctl gen config install https://10.5.1.2:6443 \
      --with-secrets secrets.yaml --with-docs=false --with-examples=false \
      --install-disk /dev/vda --output nodes/
```
```txt
generating PKI and tokens
Created nodes/controlplane.yaml
Created nodes/worker.yaml
Created nodes/talosconfig
```

* The endpoint is the Kubernetes API URL: a control plane node IP with `:6443`
* **What it produces**
  - `controlplane.yaml` — machine config for controlplane nodes.
  - `worker.yaml` — machine config for worker nodes.
  - `talosconfig` — client config pointing at the endpoint, for `talosctl`.
* **Key flags**
  - `--with-secrets secrets.yaml` — reuse the PKI from `gen secrets` (nodes share the same CAs)
  - `--install-disk` — where the OS is installed


Show the installer configuration for the control plane:

```bash
yq 'select(.kind == "UnattendedInstallConfig")' nodes/controlplane.yaml
```
```txt
apiVersion: v1alpha1
kind: UnattendedInstallConfig
installer:
  image: factory.talos.dev/metal-installer/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba:v1.14.6
provisioning:
  diskSelector:
    match: disk.dev_path == "/dev/vda"
  wipe: false
```

* The installer image comes from the **Image Factory**  
  <https://docs.siderolabs.com/talos/v1.14/learn-more/image-factory>
* `diskSelector.match` is a **Common Expression Language** (CEL)   
  <https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/storage-and-disk-management/disk-management/common#disk-selector>
  - Select a disk by size, model or serial number
  - Example `disk.size > 8u * GB && !disk.rotational`

Set the endpoint and the node:

```bash
talosctl config endpoint 10.5.1.2
talosctl config node 10.5.1.2
talosctl config info
```
```txt
Current context:     delta
Nodes:               10.5.1.2
Endpoints:           10.5.1.2
Roles:               os:admin
Certificate expires: 1 year from now (2027-11-22)
```

## Local Installation

Sends a machine config to a running node, **turns maintenance-mode ISO nodes into cluster members**

```bash
talosctl apply-config --insecure --nodes 10.5.1.2 \
      --file nodes/controlplane.yaml \
      --config-patch @patches/controlplane-1.yaml
```
```txt
Applied configuration without a reboot
```

```bash
talosctl apply-config --insecure -n 10.5.1.3 \
      --file nodes/worker.yaml \
      --config-patch @patches/worker-1.yaml
```
```txt
Applied configuration without a reboot
```

Command-line options:

- `--insecure` — talks to the maintenance API (nodes have no cluster identity yet).
- `--file ...` — machine config rendered by `gen config`
  - The node receives its machine configuration...
  - ...cluster name, endpoint, PKI references, network and install settings.
- `--config-patch @patches/...` — layers per-node overrides
- The agent validates the config ...applies the required changes ...reboots if required

```bash
export TALOSCONFIG="$PWD/nodes/talosconfig"
talosctl config endpoint 10.5.1.2
talosctl config node 10.5.1.2
talosctl config info
 ```

 ```bash
talosctl get unattendedinstallstatus -n 10.5.1.2
```
```txt
NODE       NAMESPACE   TYPE                      ID                   VERSION   PHASE
10.5.1.2   runtime     UnattendedInstallStatus   unattended-install   1         installed
```

## Bootstrap Kubernetes

**Bootstrap starts etcd on the first control plane node.**

- New control plane nodes run etcd in join mode
- They wait forever for existing etcd peers
- `talosctl bootstrap` tells one node to give up waiting
- That node creates a single-node etcd cluster
- Only then can the Kubernetes control plane start

Without it, the cluster never comes up.

```bash
talosctl bootstrap # no output equals success
# Is etcd up?
talosctl service etcd
```
```txt
NODE     10.5.1.2
ID       etcd
STATE    Running
HEALTH   OK
EVENTS   [Running]: Health check successful (1m52s ago)
         [Running]: Started task etcd (PID 133) for container etcd (1m57s ago)
         [Preparing]: Creating service runner (1m57s ago)
         [Preparing]: Running pre state (1m57s ago)
         [Waiting]: Waiting for volumes /var/lib, ETCD to be mounted, service "cri" to be "up", time sync, network, etcd spec (1m57s ago)
         [Starting]: Starting service (1m57s ago)
```

**Show the health of the core Kubernetes control plane component**:

```bash
# Is Kubernetes running on the node?
talosctl get staticpodstatus
```
```txt
NODE       *   NAMESPACE   TYPE              ID                                                         VERSION   READY
10.5.1.2   +   k8s         StaticPodStatus   kube-system/kube-apiserver-delta-controlplane-1            1         True
10.5.1.2   +   k8s         StaticPodStatus   kube-system/kube-controller-manager-delta-controlplane-1   1         True
10.5.1.2   +   k8s         StaticPodStatus   kube-system/kube-scheduler-delta-controlplane-1            1         True
```

**What are static pods?**

- apiserver, controller-manager, scheduler run as static pods
- Talos writes their manifests to disk itself (the StaticPodServerController)
- kubelet starts them directly, no controller deploys them
- One set of static pods exists per control plane node

 Get a working `kubectl` and prove the cluster is usable.

```bash
export KUBECONFIG="$PWD/nodes/kubeconfig"
talosctl kubeconfig "$KUBECONFIG"
# first real Kubernetes API call, using that credentialat 
kubectl get nodes -o wide
```
```txt
NAME                   STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE          KERNEL-VERSION          CONTAINER-RUNTIME
delta-controlplane-1   Ready    control-plane   13m   v1.37.1   10.5.1.2      <none>        Talos (v1.14.6)   6.18.63-talos (amd64)   containerd://2.3.7
delta-worker-1         Ready    <none>          13m   v1.37.1   10.5.1.3      <none>        Talos (v1.14.6)   6.18.63-talos (amd64)   containerd://2.3.7
```

What the output proves

- Both nodes are registered in the cluster API
- `Ready` — kubelet reports healthy on both
- `control-plane` role is set on the control plane node
- `INTERNAL-IP` matches the network: 10.5.1.2 and 10.5.1.3
- `OS-IMAGE` confirms both run Talos v1.14.6
- This is the end of the delta flow: OS installed, etcd up, control plane up, nodes joined

# Experiments

## Partitions & Volumes

**`get discoveredvolumes`** — raw detection: every disk and partition the node sees

- Reports what it found on the hardware: filesystem type, size, labels
- Includes the ISO drive (`sr0`) and the in-RAM SquashFS (`loop1`)




```bash
talosctl get discoveredvolumes
```
```txt
NODE       NAMESPACE   TYPE               ID      VERSION   TYPE        SIZE     DISCOVERED   LABEL       PARTITIONLABEL
10.5.1.2   runtime     DiscoveredVolume   loop1   1         disk        94 MB    squashfs                 
10.5.1.2   runtime     DiscoveredVolume   sr0     1         disk        1.1 GB                            
10.5.1.2   runtime     DiscoveredVolume   vda     1         disk        11 GB    gpt                      
10.5.1.2   runtime     DiscoveredVolume   vda1    1         partition   2.2 GB   vfat         EFI         EFI
10.5.1.2   runtime     DiscoveredVolume   vda2    1         partition   1.0 MB   talosmeta                META
10.5.1.2   runtime     DiscoveredVolume   vda3    1         partition   105 MB   xfs          STATE       STATE
10.5.1.2   runtime     DiscoveredVolume   vda4    1         partition   8.4 GB   xfs          EPHEMERAL   EPHEMERAL
```

**`get volumestatuses`** — managed state: the volumes Talos actually owns

- Shows each volume's lifecycle `PHASE` (here: `ready`)
- Shows which device backs it (`/dev/vda1-4`)

```bash
talosctl get volumestatuses
```
```txt
NODE       NAMESPACE   TYPE           ID            VERSION   TYPE        PHASE   LOCATION    SIZE
#...
10.5.1.2   runtime     VolumeStatus   EPHEMERAL     3         partition   ready   /dev/vda4   8.4 GB
#...           
10.5.1.2   runtime     VolumeStatus   META          3         partition   ready   /dev/vda2   1.0 MB
10.5.1.2   runtime     VolumeStatus   STATE         7         partition   ready   /dev/vda3   105 MB
```

**`get mountstatuses`** — mount points: where each device is attached in the filesystem

```bash
talosctl get mountstatuses
```
```txt
NODE       NAMESPACE   TYPE          ID           VERSION   SOURCE      TARGET  FILESYSTEM   VOLUME
#...
10.5.1.2   runtime     MountStatus   EPHEMERAL    5         /dev/vda4   /var    xfs          EPHEMERAL
```

The partition layout for the local disk:

- **`vda1`**, 2.2 GB, vfat, label **`EFI`** — boot partition
  - Holds GRUB, kernel images, initrd
  - Mounted at `/boot`
- **`vda2`**, 1.0 MB, talosmeta, label **`META`**
  - Key-value store, not a normal filesystem
  - Holds install state, boot mode, upgrade metadata
- **`vda3`**, 105 MB, xfs, label **`STATE`**
  - Holds the machine config and the Talos state database (all the resources)
- **`vda4`**, 8.4 GB, xfs, label **`EPHEMERAL`** — mounted at `/var`
  - Holds container images, etcd data (`/var/lib/etcd`), kubelet data
  - "Ephemeral" because it is wiped on reinstall or reset

## Comparison

This section compares the two machine configs:

- `.talos/controlplane.yaml` — auto-generated by `talosctl cluster create qemu` (the provisioner)
- `nodes/controlplane.yaml` — rendered by hand with `talosctl gen config install`

```bash
# Lists the kind of every document in each multi-doc YAML, then diffs the two lists
diff <(yq -N '.kind // "(v1alpha1)"' .talos/controlplane.yaml) \
     <(yq -N '.kind // "(v1alpha1)"' nodes/controlplane.yaml)
```
```txt
30,31d29
< LinkAliasConfig
< DHCPv4Config
```

The provisioner adds `LinkAliasConfig` and `DHCPv4Config`. Reason:

- `talosctl cluster create` runs a **userspace load balancer** on the host at the gateway IP `10.5.1.1`
- `LinkAliasConfig` gives a stable name to a network interface (no IPs involved)
- `DHCPv4Config` lets the node get its **IP from the DHCP server** (provisioner runs on the bridge)
- Your manual `gen config` has none of this, because you set a static IP in the config patch

```bash
yq 'select(.kind == "KubeClusterConfig") | .endpoint' \
      .talos/controlplane.yaml nodes/controlplane.yaml
```
```txt
https://10.5.1.1:6443
---
https://10.5.1.2:6443
```

- Provisioner: `https://10.5.1.1:6443` — the load balancer VIP, survives node loss
- Manual: `https://10.5.1.2:6443` — the raw node IP you passed to `gen config install`

Why this matters for your experiment...

- With your manual config, clients must reach the node IP directly
- The provisioner endpoint 10.5.1.1 keeps working even if the control plane node changed IP
- The two configs are otherwise identical — the delta proves both paths converge

## Maintenance Mode

```bash
talosctl -n 10.5.1.3 reset --reboot
```

* `--graceful` (default) cordons and drains the node first.
* `--wipe-mode all` (default) deletes the complete system disk. The next boot uses the ISO again.
* `--reboot` starts it again after the wipe.

```bash
kubectl get nodes
```
```txt
NAME                   STATUS                        ROLES           AGE   VERSION
delta-controlplane-1   Ready                         control-plane   40m   v1.37.1
delta-worker-1         NotReady,SchedulingDisabled   <none>          40m   v1.37.1
```

The node is in maintenance mode again

```bash
talosctl -n 10.5.1.3 version --insecure
talosctl -n 10.5.1.3 get disks --insecure
```

Install the worker again:

```bash
talosctl apply-config --insecure -n 10.5.1.3 \
      --file nodes/worker.yaml \
      --config-patch @patches/worker-1.yaml
kubectl get nodes --watch 
```
```txt
Applied configuration without a reboot
NAME                   STATUS                        ROLES           AGE   VERSION
delta-controlplane-1   Ready                         control-plane   43m   v1.37.1
delta-worker-1         NotReady,SchedulingDisabled   <none>          45m   v1.37.1
delta-worker-1         NotReady                      <none>          45m   v1.37.1
#...
delta-worker-1         Ready                         <none>          46m   v1.37.1
```

The node gets the same name and joins again. Kubernetes uses the existing Node object.

## Reset Partitions

Install some workload and check the container image:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/web
kubectl get pods -l app=web -o wide

talosctl -n 10.5.1.3 image list --namespace cri | grep nginx
```
```txt
10.5.1.3   docker.io/library/nginx:1.29-alpine                                                                  sha256:5616878291a2eed594aee8db4dade5878cf7edcb475e59193904b198d9b830de   26 MB    io.cri-containerd.image=managed                                   2026-11-22T15:55:21Z
10.5.1.3   docker.io/library/nginx@sha256:5616878291a2eed594aee8db4dade5878cf7edcb475e59193904b198d9b830de      sha256:5616878291a2eed594aee8db4dade5878cf7edcb475e59193904b198d9b830de   26 MB    io.cri-containerd.image=managed                                   2026-11-22T15:55:21Z
```

**Wipe only `EPHEMERAL` and reboot**

```bash
talosctl -n 10.5.1.3 reset --system-labels-to-wipe EPHEMERAL --reboot
```

When the node is `Ready` again, examine the result:

```bash
# watch it self-heal
kubectl get pods -l app=web -o wide --watch
```

- The old pod object is replaced by a new one (new name suffix, new IP)
- RESTARTS of the Deployment stays at 0; the pod was recreated, not restarted

**Why this heals...**

- The Deployment controller runs on the control plane, in etcd
- It compares desired state (1 replica) with actual state
- After the wipe, the pod is `Completed` or gone — both count as "not running"
- The controller creates a new pod on the node, kubelet starts it, image re-pulled

