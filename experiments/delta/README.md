---
title: Talos with Qemu
date: 2026/11/21
date-modified: 2026/11/22
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

## Install

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