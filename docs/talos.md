# Overview

What is Talos Linux?  
<https://docs.siderolabs.com/talos/v1.14/overview/what-is-talos>

* **Kubernetes-only Linux distribution**
* API managed, minimal, immutable & secure by default

**Philosophy** (of Siderolabs)  
<https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy>

* Core Principles
  - Distributed: no single points of failure; HA dataplane first
  - Immutable: always runs from signed SquashFS image; disk is wipe-safe
  - Minimal: no shell, SSH, or GNU tools; under 80 MB image
  - Ephemeral: all data is replicated or reconstructable
* Design
  - Secure: signed images, no passwords, encrypted comms, rotating certs
  - Declarative: one YAML file configures Talos and Kubernetes
  - Not based on any distro: custom userspace in Go, no systemd
  
Sidero Labs, Talos Linux, Github  
<https://github.com/siderolabs/talos>

* Licensed under the **MPL-2.0** (Mozilla Public License)
* What it give you...
  - Free to use, modify, and distribute, even commercially
  - Includes a patent grant from each contributor
  - Can combine with code under other licenses (including proprietary)
* What It Requires...
  - Copyleft is per-file: modifying file must be released under MPL-2.0
  - New files you add can be under any license
  - Keep license notices intact; provide source for modified files

## Architecture

Immutable, atomic Linux image where `machined` replaces init, components talk over local gRPC sockets

**Core Components**  
<https://docs.siderolabs.com/talos/v1.14/learn-more/components>

- `machined`: the init process; its one job is run Kubernetes
- `apid`: gRPC gateway; `talosctl` talks to it; it routes requests to other daemons
- `trustd`: root of trust; controls trust-based actions like file placement
- `containerd`: runtime; system services in system namespace, pods in k8s.io
- `networkd`, `udevd`: networking config and device handling
- Kernel hardened per the Kernel Self Protection Project (KSPP)

Design Principles

- Atomic: one signed, versioned, immutable image.
- Modular: daemons communicate only through gRPC interfaces.
- No user services: only fixed services that serve Kubernetes.
- Kubernetes runs as containers, not as host services.

**System Architecture** - one signed, immutable image, layered read-only root file system  
<https://docs.siderolabs.com/talos/v1.14/learn-more/architecture>

* Partitions with the following labels:
  - EFI and BIOS: boot loader stages
  - BOOT: initramfs and kernel
  - META: node identity
  - STATE: machine config, cluster discovery, KubeSpan data
  - EPHEMERAL: mounted at `/var`
* File-system layers:
  - Layer 1: read-only squashfs rootfs, loop-mounted into memory.
  - Layer 2: tmpfs; `/system` holds writable files bind-mounted over /etc.
  - Layer 3: overlayfs for persistent paths, for example `/etc/kubernetes`, backed by XFS at /var.
  - `/var` holds etcd, kubelet, and containerd state; wiped only on reset.

**Control Plane** Nodes  
<https://docs.siderolabs.com/talos/v1.14/learn-more/control-plane>

* Run `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`
* Tainted by default to keep workloads off them
* Prefer 3 or 5 nodes; even counts fail quorum more often
* `etcd` quorum
  - Reads and writes need a majority of members
  - Lost quorum locks the database to protect data
  - Performance drops as the cluster grows

**Endpoints & Nodes**  
<https://docs.siderolabs.com/talos/v1.14/learn-more/talosctl#endpoints-and-nodes>
* Endpoints
  - Endpoints are the addresses `talosctl` connects to directly.
  - They can be IPs, DNS names, or a load balancer.
  - Point them at the control plane nodes.
  - With several endpoints, the client load balances and fails over.
* Nodes
  - The node is the target of the API call, selected with `-n`/`--nodes`.
  - The docs recommend the flag over storing nodes in the config file.
  - Each endpoint proxies requests to other nodes. You keep one endpoint for the whole cluster.
  - Endpoint and node must join the same cluster. Proxying uses certificate auth.