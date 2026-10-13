# kit — Kubernetes Interactive Toolkit

The unified successor to `aksc` and `aksm`. One CLI, one menu, one mental model.

```
kit
```

…drops you into a fuzzy-searchable picker for **subscription → cluster → namespace → action**, then loops you back to the action menu so you can keep working without restarting.

## Why

`aksc` and `aksm` shared the same picker flow but lived as two scripts. `kit` collapses them into one binary, adds proper back-navigation between menus, pages long output through `less` so you can scroll/select/copy, and adds an `exec` action for shelling into running pods.

## Install

```bash
brew install zaidhassan168/aks-tools/kit      # install
brew upgrade zaidhassan168/aks-tools/kit      # update
kit --version                                 # check
```

Always use the full name: a plain `brew install kit` can install a different package. Activate your Azure role (for example in PIM) before you use `kit`, because it gets fresh credentials on every run.

Without Homebrew:

```bash
curl -fsSL https://raw.githubusercontent.com/zaidhassan168/aks-tools/main/install.sh | bash
```

Or manually:

```bash
git clone https://github.com/zaidhassan168/aks-tools.git
cp aks-tools/kit/kit ~/bin/kit
chmod +x ~/bin/kit
```

## Prerequisites

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) (`az`) — logged in via `az login`
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [fzf](https://github.com/junegunn/fzf) (recommended) — fuzzy finder for the menus
- `python3` (for parsing pod JSON in the `console` action)
- `less` (almost always already installed)

## Usage

### Interactive (default)

```bash
kit
```

Pick subscription → cluster → namespace → action. After each action you stay in the action menu. Use `back-ns`, `back-cluster`, or `back-sub` to hop up a level. `quit` to exit.

### Direct subcommand

```bash
kit pods       # pick sub/cluster/ns then list pods
kit logs       # pick sub/cluster/ns/pod then tail logs
kit console    # spin up a Rails console pod
kit restart    # rollout restart a deployment
kit kv-set     # change a Key Vault setting
kit --version  # print the kit version
```

### Shortcuts

```bash
kit pods --env prd --ns myapp     # --env sets subscription and cluster together
```

In the namespace picker, the namespace you used last on that cluster is first.

### Admin

```bash
kit cleanup           # pick a cluster, delete your orphaned Rails console pods there
kit list-orphaned     # list them without deleting
kit cleanup someone   # delete someone else's leaked pods
```

### Help

```bash
kit --help
```

## Actions

| Action | What it does |
| --- | --- |
| `pods` | List pods (paged) |
| `logs` | View logs of a picked pod (paged, tail 200) |
| `logs-f` | Follow logs live |
| `describe` | Describe a pod (paged) |
| `events` | Recent namespace events (paged) |
| `exec` | Open a shell in a running pod |
| `console` | Spin up a Rails console pod (mirrors `aksc`) |
| `cm-view` | Show a ConfigMap as full YAML |
| `cm-get` | Show ConfigMap data only |
| `cm-set` | Add/update ConfigMap values interactively |
| `cm-del` | Delete a ConfigMap key |
| `kv-get` | Show the env values that a Key Vault chart syncs into the pods' secret |
| `kv-set` | Change existing Key Vault values (see below) |
| `restart` | Rollout-restart a picked deployment |
| `restart-all` | Rollout-restart every deployment in the namespace |

`cm-set` and `cm-del` warn you when no workload in the namespace reads that ConfigMap, for example after an app moved to Key Vault.

## Key Vault settings (`kv-get`, `kv-set`)

Apps that moved to Azure Key Vault no longer read a ConfigMap. Their Helm chart has a `SecretProviderClass` (SPC): a list of Key Vault secret names. On each deploy, Helm writes that list to the cluster. The Secrets Store CSI driver then fetches only the listed secrets and copies them into a Kubernetes secret, for example `myapp-secret`. The pods read their env from that secret when they start.

`kit kv-set`:

1. Reads the SPC in the namespace and lists only the keys that already exist.
2. Lets you choose one or more keys (Tab in fzf), shows each current value, and asks for the new value.
3. Shows old and new values and asks you to confirm.
4. Writes the values to Key Vault. You need the **Key Vault Secrets Officer** role on that vault.
5. Waits until the CSI driver syncs the new values (it checks about every 2 minutes).
6. Offers to restart the deployments that read the secret. Env vars are read only at pod start.

**A new key needs a chart PR.** If you type a key that is not in the list, `kv-set` tells you the Key Vault name to create and the line to add to the app's SecretProviderClass template. Helm rebuilds the list from git on every deploy, so a key that is only added in the cluster disappears at the next deploy.

## How `console` works

1. Detects the app's container image from the namespace's deployments.
2. Uses the same env source as a running app pod: the Key Vault secret for Key Vault apps, the ConfigMap for the others.
3. Copies scheduling (service account, node selector, tolerations, affinity) from a real Running pod so the console lands on the right node pool.
4. Creates a one-off `rails-console-<user>-<ts>` pod, attaches your terminal to it.
5. **Deletes the pod immediately when you exit** — no leaked pods after a clean exit.
6. If kit cannot delete it (closed laptop, VPN drop), Kubernetes stops the pod after `KIT_CONSOLE_MAX_HOURS` (default 12).

If a pod fails to start, kit prints the pod debug (describe + events + container logs) automatically. Set `KEEP_POD=1 kit console` to keep failed pods for further inspection.

If `attach` doesn't show a Rails prompt (terminal compatibility quirk), use `KIT_ATTACH_MODE=exec kit console` — it sleeps the pod and `exec`s a fresh console process inside it.

## Credentials

`kit` gets fresh cluster credentials on every run, so your PIM role must be active. It keeps them only in a private temp kubeconfig (`kit-<sub-prefix>-<cluster-name>` contexts) and deletes that file when it exits. It also tidies up the `kit-*` and `aksc-*` contexts that older versions added to `~/.kube/config`.

`kit cleanup` and `kit list-orphaned` ask for a cluster first, then work on that cluster.

## VPN (macOS)

The cluster API servers sit behind privatelink, so `kit` manages the Azure VPN for you:

- **VPN already connected** when kit starts: left alone, and *not* disconnected on exit.
- **VPN disconnected**: kit connects it (via `scutil --nc start`), waits for `Connected`, then disconnects it again when kit exits — normal quit, Ctrl-C, or an error.
- The service is auto-detected from `scutil --nc list` (first enabled Azure VPN Client entry). Override with `KIT_VPN_SERVICE`, or disable the whole thing with `KIT_VPN=0`.

## Environment variables

| Var | Effect |
| --- | --- |
| `KEEP_POD=1` | Don't delete the Rails console pod if it failed to start (for debugging) |
| `KIT_ATTACH_MODE=exec` | Default the `console` action to exec mode instead of attach |
| `KIT_CONSOLE_MAX_HOURS=12` | Hours after which Kubernetes stops a console pod |
| `KIT_EXTENSIONS=path` | Private extension file (default `~/.config/kit/extensions.sh`, see `kit --help`) |
| `KIT_VPN=0` | Never touch the VPN |
| `KIT_VPN_SERVICE=name` | VPN service name from `scutil --nc list` (default: Azure VPN Client) |
| `KIT_VPN_KEEP=1` | Leave the VPN up on exit even if kit connected it |
| `KIT_VPN_TIMEOUT=45` | Seconds to wait for the VPN to connect before continuing |
