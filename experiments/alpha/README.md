---
title: Talos 101
date: 2026/11/19
date-modified: 2026/11/20
---

# Deployment

Builds a full 3-node Talos Kubernetes cluster as Docker containers

```bash
talos_version=$(talosctl version --client | awk '/Tag:/ {print $2; exit}')
export TALOSCONFIG="$PWD/.talos/talosconfig"
talosctl cluster create docker \
  --name alpha --subnet "10.5.1.0/24" \
  --state "$PWD/.talos/clusters" \
  --talosconfig-destination "$TALOSCONFIG" \
  --image "ghcr.io/siderolabs/talos:$talos_version" \
  --workers 2
# Do not Ctrl-C, this leaves you without the final kubeconfig
```

Hint: Make sure the 10.5.1.0/24 network is not routed to a another network (by `sshuttle` for example)

Command-line options:

* `docker` — selects the Docker provisioner (alternative: qemu)
* `--name alpha `— cluster identity, Docker network name, container name prefix
* `--subnet 10.5.1.0/24` — CIDR of the internal Docker network
* `--state ...` — where `state.yaml` is stored
* `--talosconfig-destination` — where the client credential file is written
* `--workers 2` — two worker; there is always exactly one control plane in Docker mode

```bash
talosctl cluster destroy --name alpha --state "$PWD/.talos/clusters"
rm -rf "$PWD/.talos"
```

# Configuration

Check the configuration after deployment:

```bash
talosctl config info
```
```txt
Current context:     alpha
Nodes:               not defined
Endpoints:           127.0.0.1:43931
Roles:               os:admin
Certificate expires: 1 year from now (2027-11-20)
```

- `Current context: alpha` — the active entry in your talosconfig
- `Nodes: not defined` — no default target node. Commands that need a node will fail unless you pass `--nodes`.
- `Endpoints: 127.0.0.1:43931` — the address `talosctl` connects to. Docker maps this host port to port 50000 (the Talos API) of `alpha-controlplane-1`. Note it changed each time the cluster is recreated.
- `Roles: os:admin` — client certificate, full admin rights on the Talos API.

## Endpoint

The client can reach the cluster through the endpoint, but it does not know which node to act on. Set a **permanent default for the endpoint**:

```bash
talosctl config node 10.5.1.2        # permanent default for this context
talosctl health --nodes 10.5.1.2     # per-command, docs-recommended
```

`10.5.1.2` is the control plane container IP, as seen from inside the Docker network.

Show the containers and the port mappings:

```bash
docker ps --filter label=talos.cluster.name=alpha --format '{{.Names}}\t{{.Ports}}'
```
```txt
alpha-worker-2  
alpha-worker-1  
alpha-controlplane-1    127.0.0.1:42629->6443/tcp, 127.0.0.1:43931->50000/tcp
```

## Services

Shows each node's IP, type and version (needs healthy nodes)

```bash
talosctl get members
```
```txt
NODE       NAMESPACE   TYPE     ID                     VERSION   HOSTNAME               MACHINE TYPE   OS                ADDRESSES
10.5.1.2   cluster     Member   alpha-controlplane-1   1         alpha-controlplane-1   controlplane   Talos (v1.14.6)   ["10.5.1.2"]
10.5.1.2   cluster     Member   alpha-worker-1         1         alpha-worker-1         worker         Talos (v1.14.6)   ["10.5.1.3"]
10.5.1.2   cluster     Member   alpha-worker-2         1         alpha-worker-2         worker         Talos (v1.14.6)   ["10.5.1.4"]
```

Show the services of the control plane node:

```bash
talosctl services
```
```txt
NODE       SERVICE      STATE     HEALTH   LAST CHANGE    LAST EVENT
10.5.1.2   apid         Running   OK       1h12m48s ago   Health check successful
10.5.1.2   containerd   Running   OK       1h12m49s ago   Health check successful
10.5.1.2   cri          Running   OK       1h12m48s ago   Health check successful
10.5.1.2   etcd         Running   OK       1h11m34s ago   Health check successful
10.5.1.2   kubelet      Running   OK       1h10m30s ago   Health check successful
10.5.1.2   machined     Running   OK       1h12m49s ago   Health check successful
10.5.1.2   trustd       Running   OK       1h12m48s ago   Health check successful
```

Show the services of a worker:

```bash
talosctl -n 10.5.1.3 services
```
```txt
NODE       SERVICE      STATE     HEALTH   LAST CHANGE    LAST EVENT
10.5.1.3   apid         Running   OK       1h14m38s ago   Health check successful
10.5.1.3   containerd   Running   OK       1h14m41s ago   Health check successful
10.5.1.3   cri          Running   OK       1h14m40s ago   Health check successful
10.5.1.3   kubelet      Running   OK       1h12m24s ago   Health check successful
10.5.1.3   machined     Running   OK       1h14m41s ago   Health check successful
```

Show the log of a service and the kernel log:

```bash
talosctl logs etcd | tail -5
talosctl -n 10.5.1.3 logs kubelet | tail -5
talosctl dmesg | tail -5
```

## Container


Talos has two containerd namespaces: `system` (Talos services) and `cri` (Kubernetes pods).

```bash
talosctl containers
```
```txt
NODE       NAMESPACE   ID       IMAGE   PID   STATUS
10.5.1.2   system      apid             128   RUNNING
10.5.1.2   system      trustd           129   RUNNING
```

The `IMAGE` column is empty: `apid` and `trustd` come from the root file system of Talos, not from an image.

Kubernetes pods running on the control plane node (`cri` namespace)

```bash
talosctl containers --namespace cri
```

Each pod shows two rows:

- **Sandbox row** (top level): image `pause:3.10.2`, status `SANDBOX_READY`. This is the pod sandbox — an empty placeholder that holds the pod's network namespace. All pod containers share its network.
- **Container rows**: the real workload containers inside that sandbox, status `CONTAINER_RUNNING`.

What runs...

- CoreDNS (two replicas — the default `replicaCount: 2`).
- kube-apiserver, kube-controller-manager, kube-scheduler (static pods).
- flannel (CNI) and kube-proxy (DaemonSets).

## Dashboard

Interactive dashboard,  built-in console UI, live node state:

```bash
# Open the dashboard 
talosctl dashboard -n 10.5.1.2,10.5.1.3,10.5.1.4
```

- The dashboard shows CPU, memory, processes and the log of each node. - Use the left and right arrow keys to change the node. Press q to stop.
- **Key use case**
  - Diagnose a node when the network is broken and `talosctl` cannot reach it.
  - In your Docker lab you cannot use the video console. talosctl dashboard is the equivalen

# Experiments

## No Shell

Direct shell in the container is not possible:

```bash
docker exec -it alpha-controlplane-1 /bin/sh
```
```txt
OCI runtime exec failed: exec failed: unable to start container process: exec: "/bin/sh": stat /bin/sh: no such file or directory
```

**Read files through the API**, `ls` and `read` work on all nodes:

```bash
talosctl ls /var/lib
talosctl read /etc/os-release
```

## Commands

**Controll-plane endpoint sends the request to many nodes**:

```bash
talosctl -n 10.5.1.2,10.5.1.3,10.5.1.4 get machinetype
```
```txt
NODE       NAMESPACE   TYPE          ID             VERSION   TYPE
10.5.1.2   config      MachineType   machine-type   2         controlplane
10.5.1.3   config      MachineType   machine-type   2         worker
10.5.1.4   config      MachineType   machine-type   2         worker
```

**Request to a worker as endpoint**:

```bash
talosctl -e 10.5.1.3 -n 10.5.1.3 get machinetype
```

The worker apid answers directly:

- Any node can be an endpoint.
- Use control plane nodes as endpoints (only they can send requests on to all other nodes)

## Watch Resources

**Watch the Kubernetes node status resource**:

```bash
# dedicated terminal, Ctrl-C to stop
talosctl -n 10.5.1.3 get nodestatus --watch
```
```txt
NODE       *   NAMESPACE   TYPE         ID               VERSION   READY   UNSCHEDULABLE
10.5.1.3   +   k8s         NodeStatus   alpha-worker-1   5         true    false
10.5.1.3       k8s         NodeStatus   alpha-worker-1   6         true    true
10.5.1.3       k8s         NodeStatus   alpha-worker-1   7         true    false
```

Test by cordon a worker:

```bash
kubectl cordon alpha-worker-1
kubectl uncordon alpha-worker-1
```

* **Cordon** — Marks the node **unschedulable**.
  - The scheduler places no new pods on it.
  - Pods already running keep running. Nothing is moved.
  - Use before maintenance: drain first, then cordon.
* **Uncordon** — Removes the mark.
  - The node accepts new pods again.
  - It is the exact reverse of cordon.
* **Drain** (related)
  - Cordon alone leaves old pods running.
  - `kubectl drain <node>` **cordons and evicts** the pods.
  - Order for maintenance: `drain`, work, then `uncordon`.

