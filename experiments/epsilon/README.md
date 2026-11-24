
# Deployment

Set the environment variables and provision the cluster:
```bash
# Set the environment
talos_version=v1.14.5
kubernetes_version=1.37.0
export TALOSCONFIG="$PWD/.talos/talosconfig"
export KUBECONFIG="$PWD/.talos/kubeconfig"

# Start the cluster
sudo --preserve-env=HOME,TALOSCONFIG,KUBECONFIG \
  "$(command -v talosctl)" cluster create qemu \
  --name epsilon --cidr "10.5.5.0/24" --workers 1 \
  --state "$PWD/.talos/clusters" \
  --talos-version "$talos_version" \
  --kubernetes-version "$kubernetes_version"
```

* **`--talos-version`**
  - Select a minor release of Talos before latest stable:  
<https://github.com/siderolabs/talos/releases>
  - For example current is v1.14.6, select v1.14.5
* **`--kubernetes-version`**
  - Select a minor release of Kubernetes before the Talos default
  - `talosctl cluster create qemu --help | grep kubernetes-version`
  - For example current is 1.37.1, select 1.37.0

Add it to the trusted zone
```bash
bridge=$(grep '^bridgename:' "$PWD/.talos/clusters/epsilon/state.yaml" | awk '{print $2}')
sudo firewall-cmd --zone=trusted --add-interface="$bridge"
```

Later tear down of the cluster
```bash
sudo --preserve-env=HOME \
      "$(command -v talosctl)" cluster destroy \
            --name epsilon --state "$PWD/.talos/clusters"
sudo rm -rf "$PWD/.talos/"
```

Summary of the cluster state 

```bash
talosctl cluster show \
      --provisioner qemu --name epsilon --state "$PWD/.talos/clusters/"
```

