# aks-tools

Interactive CLI tools for Azure Kubernetes Service (AKS). Stop memorising long `kubectl` invocations, stop juggling `az account set`, stop copy-pasting namespace names.

## `kit` — Kubernetes Interactive Toolkit

The recommended tool. One command, fuzzy menus for **subscription → cluster → namespace → action**, with proper back-navigation between menus.

```bash
kit
```

What you get in one CLI:

- **Pods, logs, describe, events** — paged through `less` so you can scroll, select, and copy.
- **`exec`** — shell into a running pod.
- **`console`** — spin up a Rails console pod (image, ConfigMap, and node scheduling auto-detected from a real running pod). Pod is **deleted the moment you exit**.
- **ConfigMap** — view full YAML, view data only, set values via `KEY=VALUE` prompts, delete keys.
- **`restart`** — rollout-restart a picked deployment and watch the rollout.
- **`cleanup` / `list-orphaned`** — sweep leaked Rails console pods across every cluster you've touched.

Each cluster gets its own `kit-<sub>-<cluster>` kube context — dev/stg/prod credentials never overwrite each other.

→ Full docs: [`kit/README.md`](./kit/README.md)

## Install

### Homebrew (recommended on macOS)

```bash
brew install zaidhassan168/aks-tools/kit
```

### One-line installer (curl)

```bash
curl -fsSL https://raw.githubusercontent.com/zaidhassan168/aks-tools/main/install.sh | bash
```

### `make install`

```bash
git clone https://github.com/zaidhassan168/aks-tools.git
cd aks-tools
sudo make install              # installs to /usr/local/bin
# or:
make install PREFIX=$HOME      # installs to $HOME/bin
```

### Manual

```bash
git clone https://github.com/zaidhassan168/aks-tools.git
cp aks-tools/kit/kit ~/bin/kit && chmod +x ~/bin/kit
```

## Prerequisites

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) (`az`) — `az login` first.
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [fzf](https://github.com/junegunn/fzf) (recommended) — fuzzy menus. Without it, `kit` falls back to numbered prompts.
- `python3` — used by `kit console` for parsing pod JSON.
- `less` — almost always already installed.

## Why

Working across multiple AKS clusters and subscriptions usually looks like this:

```bash
az account set --subscription "..."
az aks get-credentials --resource-group "..." --name "..."
kubectl config use-context "..."
kubectl -n some-long-namespace-name get pods
kubectl -n some-long-namespace-name logs pod-name-hash-xyz --tail=200 | less
```

`kit` collapses all of that into picker menus and remembers each cluster as its own context.

## Legacy commands

The repo still ships [`aksc`](./aksc/) and [`aksm`](./aksm/) as **thin shims** that forward to `kit`, so existing muscle memory keeps working. New users should reach for `kit` directly.

| Legacy | Forwards to |
| --- | --- |
| `aksc` | `kit console` |
| `aksc --exec` | `KIT_ATTACH_MODE=exec kit console` |
| `aksc --list` / `aksc --cleanup-all` | `kit list-orphaned` / `kit cleanup` |
| `aksm <action>` | `kit <action>` |

## License

MIT — see [`LICENSE`](./LICENSE).
