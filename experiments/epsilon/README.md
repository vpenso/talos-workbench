
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

# Concepts

### Talos Releases

Supported Kubernetes versions and platforms per Talos version  
<https://docs.siderolabs.com/talos/v1.14/getting-started/support-matrix>
* Talos minor releases ship every 4 months, aligned with the Kubernetes cycle
* Each Talos minor has a **default Kubernetes** version (Talos 1.14 ships 1.37).
* Each Talos minor supports **six Kubernetes minors**, counted back from its default.
* The versions are **independent**: a Talos upgrade does not change Kubernetes.
* To raise Kubernetes past the supported range, upgrade Talos first.
* Migrations are tested only between adjacent minor versions.

### Image Factory

Image Factory documentation, source and UI:  
<https://docs.siderolabs.com/talos/v1.14/learn-more/image-factory>  
<https://github.com/siderolabs/image-factory>  
<https://factory.talos.dev>

* Frontends: HTTP download, PXE, and container registry.
* A **model** is one image. Inputs: schematic ID, Talos version, model type, architecture.
* The installer image serves install and upgrade...
  - ...carries only system extensions
  - ...kernel args and META apply only to boot media.
* Use the same **schematic** for boot media and installer image...
  - ...or the extensions change at first install.
* Built on the Talos `imager` container; all images are signed. It can also run on-premises.

**Five asset classes**
* **ISO** — bootable install media for install to disk.
* **kernel, initramfs, kernel command line**...
  - ...components required for network boot (PXE)...
  - ...script at `pxe.factory.talos.dev` references the kernel and initramfs over HTTP.
* **UKI** — a **Unified Kernel Image**...
  - ...kernel, initramfs, and command line packed into one EFI file
  - ...for UEFI boot, especially Secure Boot.
* **Disk images** — prebuilt OS disk, ready to attach
* **Installer image** — installs Talos to disk (also performs upgrades)

### Schematic

* A **schematic** is a small YAML file with the customizations of a Talos image:
  - `systemExtensions`,
  - `extraKernelArgs`,
  - `meta` values
  - `overlay` (for single board computers)
* It is **content-addressable**: the ID is a hash of the content.
  - The same file always gives the same ID.
  - The vanilla schematic has ID `3765...b4ba`.
