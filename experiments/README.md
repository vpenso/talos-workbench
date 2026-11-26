# Experiments

The individual experiments in this directory, in order:

| Experiment | Description |
| ---------- | ----------- |
| [Alpha](alpha/README.md) | Talos 101: a 3-node Talos Kubernetes cluster in Docker containers |
| [Beta](beta/README.md) | Applying config patches to a 3-node Docker cluster, including **try mode**, etc-files, and DNS-over-TLS |
| [Gamma](gamma/README.md) | A 3-node Docker cluster running **Cilium** as the only CNI and service load balancer |
| [Delta](delta/README.md) | A local cluster on **QEMU**: ISO boot, manual install, partition layout, and node reset flows |
| [Epsilon](epsilon/README.md) | Talos and Kubernetes upgrade on a two-node QEMU cluster, OS rollback, and re-upgrade, with the booted **UKI** entry verified at each step |
| [Zeta](zeta/README.md) | A three-control-plane QEMU cluster with a **Layer 2 VIP**: API reachability during VIP failover and a hung node, etcd maintenance, control plane replacement, and snapshot recovery |
