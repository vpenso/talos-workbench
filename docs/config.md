
# Talos Config

**`talosconfig` file is talosctl's client configuration**

Generation and location:

- Created by `talosctl gen config` alongside `controlplane.yaml`/`worker.yaml`.
- Default path: `~/.talos/config`.
- Override with `--talosconfig <path>` or the `TALOSCONFIG` environment variable

Direct manual editing is discouraged; use the CLI.

```bash
# Client-side connection setup for the active context
talosctl config info

# Address to the active context's endpoints
talosctl config endpoint $ip ...

# Set the target node(s)
talosctl config node $node0 ...
```

What each does

- `endpoints` — Addresses `talosctl` connects to for the Talos API (port 50000).
- `nodes` — Default targets for commands. They do not have to be endpoints.

## Machine Configuration  

**Required Inputs**
- `secrets.yaml`, generated once at cluster creation
- Patch files describing differences from defaults
- Cluster name and Kubernetes endpoint
- `--kubernetes-version` and `--talos-version`, kept fixed

**Workflow**
- `talosctl gen config` with secrets, versions, and patches
- Apply the generated configs to the nodes
- Discard the output. Never store generated configs in Git
- **Store only patches and `secrets.yaml` in Git**
- Keep in mind...
  - Changing `--talos-version` may add new fields or defaults.
  - This can silently change cluster behavior.
  - Only bump it when upgrading Talos is intended.
  - Intent stays in small patches; versions live in regenerated bases.

## Patches

**Patche- vs Config-Files**
| Aspect | Patch files | Config files |
|---|---|---|
| Content stored in Git | Small diffs from defaults | Entire generated config |
| Size | Small, focused | Large, mostly boilerplate |
| Readability | Shows only your intent | Hard to find your changes |
| talosctl upgrades | Patches stay stable | Regeneration changes defaults, file goes stale |
| Drift risk | Low: base always regenerated | High: file embeds old defaults |
| Reproducibility | Same inputs, same output | Depends on which version generated the file |
| Review in Git | Easy: only real changes | Noisy: unrelated fields change |
| Editing live nodes | Can patch running nodes | Must regenerate and re-apply |
| Merge conflicts | Rare | Common |

A patch file is a **partial machine config**...
* ...one or more YAML documents that merge into the full config.
* **Format**
  - Same `apiVersion`/`kind` style as machine config documents.
  - A file holds many documents, separated by `---`.
  - Each document merges into the matching document.
  - Missing documents are appended to the config.
* **Merge Rules**
  - Scalar values: patch overrides the base value.
  - Lists: patch items are appended, usually.
  - Exceptions: pod/service subnets are replaced, not appended.
  - Interfaces merge on `interface` or `deviceSelector` keys.
  - `$patch: delete` removes a field or a whole document.