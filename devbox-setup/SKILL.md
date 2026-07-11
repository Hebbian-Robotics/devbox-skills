---
name: devbox-setup
description: Set up a shared cloud dev VM ("devbox") with zero public ports — Tailscale + Tailscale SSH for identity-based auth, mosh + tmux for sessions that survive laptop sleep and reboots, working clipboard through mosh, and a cloud-native break-glass path (GCP IAP or AWS SSM). Use when provisioning a team devbox, onboarding a teammate onto one, debugging devbox connection/clipboard/session problems, or pasting local screenshots/images into a remote agent session.
---

# Cloud devbox: roaming-proof access with no public ports

A recipe for a shared Linux VM that a small team can use as a persistent dev machine:

- **No public SSH.** Nothing listens to the internet; day-to-day access rides Tailscale.
- **Identity auth.** Tailscale SSH means no key management — your tailnet login is your credential.
- **Sessions survive everything.** mosh handles laptop sleep / Wi-Fi roaming; tmux + resurrect
  survive VM reboots; copy/paste works through the whole stack.

The recipe is cloud-agnostic — anything that runs a Linux VM works. Provisioning and break-glass
are the only provider-specific parts; GCP and AWS variants are given below.

Fill in these placeholders throughout: `<NODE>` (tailnet machine name, e.g. devbox), `<USER>`
(your unix user), plus the provider identifiers in section 1.

Personal things stay personal: each person syncs their **own** credentials, dotfiles, and API keys
to their own unix account on the VM. Never copy another person's `~/.ssh`, tokens, or `.env` files.

## 1. Provision + harden (admin, once per VM)

Use on-demand/standard provisioning (not spot/preemptible) so the VM isn't reclaimed mid-session.
The goal state is identical on every cloud: **a verified keyboard-independent break-glass path
first, then zero internet-facing ports.**

### GCP (GCE)

```sh
# Pin the external IP so a stop/start never breaks anything (promote in-place, no downtime):
gcloud compute addresses create <VM>-static --addresses <CURRENT_EXTERNAL_IP> --region <REGION>

# Break-glass BEFORE closing the front door — IAP-only SSH rule, then PROVE it works:
gcloud compute firewall-rules create allow-ssh-iap --network <NETWORK> \
  --allow tcp:22 --source-ranges 35.235.240.0/20   # Google IAP range
gcloud compute ssh <VM> --zone <ZONE> --tunnel-through-iap --command 'echo iap-ok'

# Only then close public SSH (first check what else shares the network —
# every VM on it loses public 22):
gcloud compute firewall-rules delete <RULE_ALLOWING_PUBLIC_22>
```

### AWS (EC2)

```sh
# Optional: pin the public IP (Tailscale makes this less important):
aws ec2 allocate-address && aws ec2 associate-address --instance-id <INSTANCE_ID> --allocation-id <ALLOC_ID>

# Break-glass with ZERO inbound rules — SSM Session Manager. The instance needs an
# instance profile with the AmazonSSMManagedInstanceCore policy (SSM agent is preinstalled
# on Amazon Linux and Ubuntu AMIs). PROVE it works before touching the security group:
aws ssm start-session --target <INSTANCE_ID>
# (scp/ssh over SSM also works via the AWS-StartSSHSession ProxyCommand in ~/.ssh/config.)

# Only then remove public SSH ingress from the security group:
aws ec2 revoke-security-group-ingress --group-id <SG_ID> --protocol tcp --port 22 --cidr 0.0.0.0/0
```

### Packages + Tailscale on the VM (all clouds)

```sh
sudo apt-get update && sudo apt-get install -y mosh tmux git   # mosh must exist on BOTH ends
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname <NODE>      # prints a login URL — approve it in a browser
sudo tailscale set --ssh=true            # identity-based SSH; add --accept-risk=lose-ssh if connected via tailnet
```

Name nodes so they scale — `gcp-devbox`, `aws-devbox`, not just `devbox` (rename later with
`sudo tailscale set --hostname <new>`).

No inbound firewall rule is needed for Tailscale — WireGuard punches out on UDP and falls back
to TCP-443 relays, so it works even from UDP-hostile networks (mosh's UDP rides inside the tunnel).

Optional cost saver: if nobody runs overnight jobs, add an auto-stop schedule (GCP instance
schedules / AWS EventBridge Scheduler) — with a pinned IP, a stopped VM restarts with nothing broken.

## 2. Onboard a person (admin, once per person)

1. Unix account on the VM: `sudo adduser <USER> && sudo usermod -aG sudo <USER>`
2. Tailnet SSH ACL (login.tailscale.com/admin/acls) — the default rule only lets owners onto
   their own devices, and its `check` action forces a browser re-auth every ~12h, which hangs
   unattended/scripted ssh. Grant each person explicit access with `accept`:
   ```json
   "ssh": [{"action": "accept", "src": ["<person@tailnet-login>"], "dst": ["<NODE's tailnet IP>"], "users": ["<USER>"]}]
   ```
3. Break-glass IAM: on GCP, **Compute Instance Admin** + **IAP-secured Tunnel User**;
   on AWS, an IAM identity allowed `ssm:StartSession` on the instance.

## 3. Your laptop (each person)

```sh
brew install mosh
brew install --cask tailscale 2>/dev/null; open -a Tailscale
```

Log into the Tailscale menu-bar app with your team account and enable **Start on login**
(if Tailscale is off, the devbox is unreachable — that's the tradeoff for zero public ports).

`~/.ssh/config` — Tailscale SSH does the auth, so no key file is needed:

```
Host <NODE>
    HostName <NODE's tailnet IP or MagicDNS name>
    User <USER>
    ServerAliveInterval 30
    ServerAliveCountMax 4
    # Reuse one connection for repeated ssh commands (~8x faster for scripts/agents)
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

(Add `ForwardAgent yes` only if you need it and trust everyone with root on the VM — a root
user on a shared machine can borrow a forwarded agent while you're connected.)

Test with `ssh <NODE> 'echo ok'`. If it hangs printing a login.tailscale.com URL, that's the ACL
`check` action — open the URL once, then get the ACL changed to `accept` (step 2.2).

## 4. tmux that survives everything (each person, on the VM)

`~/.tmux.conf` — three clipboard lines matter here; each covers a different copy path
(see "Clipboard reality check" below for what works over which transport):

```
set -g mouse on
set -g history-limit 100000
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:Tc"
setw -g mode-keys vi
set -g escape-time 10
set -g renumber-windows on
# Advertise OSC 52 clipboard to tmux with an explicit "c" target + BEL terminator
# (the most widely accepted form) so copy-mode yanks reach the outer terminal:
set -as terminal-overrides ',xterm*:Ms=\E]52;c;%p2%s\007'
# Let applications inside panes (interactive CLIs with "press c to copy" etc.) write the
# clipboard too — the default "external" only forwards tmux's own copy-mode, so app-initiated
# copies claim success but never arrive:
set -g set-clipboard on
# Some interactive CLIs (AI coding agents etc.) wrap their OSC 52 in DCS passthrough —
# without this tmux silently drops those copies even though the app reports success:
set -g allow-passthrough on
# Sessions survive VM reboots: resurrect saves layouts, continuum autosaves/restores.
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @continuum-restore 'on'
set -g @resurrect-capture-pane-contents 'on'
run '~/.tmux/plugins/tpm/tpm'
```

```sh
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
tmux start-server; tmux source-file ~/.tmux.conf 2>/dev/null
~/.tmux/plugins/tpm/bin/install_plugins    # must run AFTER the server sources the config
```

## 5. Daily driver

```sh
mosh <NODE>
```

That's the entry point: a plain mosh shell. It survives lid-close, Wi-Fi switches, and IP
changes on its own — tmux is a second, optional layer you attach when you want work to outlive
the connection itself (VM reboots, multi-day jobs, picking the same shell up from another machine):

```sh
tmux new -A -s main      # inside the mosh shell; ctrl-b d detaches, everything keeps running
```

Keeping tmux manual also sidesteps a real tradeoff: tmux intercepts the terminal, so interactive
apps behave slightly differently inside it (clipboard, mouse). The config above fixes the known
cases, but a plain mosh shell is always the most transparent path — use it by default, attach
tmux when persistence matters.

### Clipboard reality check (read this before debugging copy/paste)

Escape-sequence copying (OSC 52 — what tmux copy-mode and "press c to copy" CLIs emit)
**does not survive stock mosh**: mosh re-renders terminal state rather than passing bytes
through, and clipboard forwarding has been an open request since 2015
(mobile-shell/mosh#637 — the PRs were never merged; claims that a `c;` Ms override fixes it
apply to patched builds). It **does** survive plain ssh untouched.

Practical rules:
- **Copy-heavy session (agent CLIs, yanking from tmux history)** → use an ssh tab:
  `ssh <NODE>` is stable over the tailnet, and with the tmux config above every copy path
  works (copy-mode, app OSC 52, DCS-wrapped).
- **Roaming/flaky-network session** → mosh, and copy visible text with your terminal's
  native selection (hold **Option** on macOS to bypass tmux/app mouse capture). For scrolled-
  back content: page through `ctrl-b [` and select screenful-wise.
- Want both at once? **Eternal Terminal** (`et`) survives roaming like mosh but is
  byte-transparent like ssh, so OSC 52 works through it.
- Config changes only apply to tmux clients attached after the config loaded — when in
  doubt, detach/reattach once.

- **Scrollback inside tmux**: `ctrl-b [` (mosh has no native scrollback; tmux's history is it).
- Worth an alias: `alias dev='mosh <NODE>'`
- **Dev servers need no tunnels**: anything listening on the VM is privately reachable from your
  laptop at `http://<NODE>:<port>` over the tailnet. `tailscale serve` adds HTTPS or shares it
  with teammates; nothing is exposed to the internet.

### Pasting screenshots (local clipboard → remote)

The reverse direction has a harder limit: terminals paste **text only**, so an image on your
local clipboard can never reach the VM through ssh or mosh — OSC 52 has no image support, and
kitty's OSC 5522 extension that would add it isn't implemented by other terminals yet. Agent
CLIs' Ctrl+V image paste reads the OS clipboard, which doesn't exist on a headless VM.

Ship the image as a file and paste its **path** instead. macOS: `brew install pngpaste`, then
in `~/.zshrc`:

```sh
# Screenshot to clipboard (Cmd+Shift+Ctrl+4) → `pshot` → Cmd+V the path into the remote CLI
# (agent CLIs read image files by path):
pshot() {
  local f="shot-$(date +%H%M%S).png"
  pngpaste "/tmp/$f" || { echo "no image on clipboard" >&2; return 1; }
  scp -q "/tmp/$f" <NODE>:/tmp/ &&
    printf '/tmp/%s' "$f" | pbcopy &&
    echo "→ <NODE>:/tmp/$f (path copied to clipboard)"
}
```

The ControlMaster block in section 3 makes the scp near-instant. Transparent Ctrl+V bridges
exist (cc-clip, clipaste: a local daemon + `ssh -R` tunnel + fake xclip on the VM), but mosh
doesn't carry SSH port forwards, so they'd need a separate plain-ssh session held open — the
path helper is the simpler fit for this stack.

## 6. Recommended developer toolchain (once per VM)

A baseline that makes the devbox feel like a real dev machine, not a bare VM. All of it is
system-wide or per-user-trivial; install once, everyone benefits:

```sh
# Core CLI tools (apt names differ from the binaries: fd-find→fd, install ripgrep→rg):
sudo apt-get update && sudo apt-get install -y \
  ripgrep fd-find jq unzip zip build-essential pkg-config libssl-dev htop
sudo ln -sf "$(command -v fdfind)" /usr/local/bin/fd   # Ubuntu names the binary fdfind

# GitHub CLI (repo auth, PRs, CI from the box):
sudo mkdir -p -m 755 /etc/apt/keyrings && \
  curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg >/dev/null && \
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list >/dev/null && \
  sudo apt-get update && sudo apt-get install -y gh
```

Per-user language/package managers — prefer the official installers over apt (apt versions
lag badly for these), and prefer the manager over the raw runtime (uv installs pythons,
fnm/corepack install nodes):

```sh
# Python: uv (replaces pip/pyenv/venv workflows)
curl -LsSf https://astral.sh/uv/install.sh | sh
# Node: fnm + corepack gives you node/pnpm/yarn pinned per-project
curl -fsSL https://fnm.vercel.app/install | bash
fnm install --lts && corepack enable pnpm
# Rust (skip unless you build Rust):
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
```

### Faster Rust builds (sccache + mold)

If the team builds Rust, two cheap, high-leverage speedups — complementary, since they hit
different phases of the build:

- **sccache** caches compiled crates so unchanged dependencies never recompile. Install
  (`cargo install sccache --locked`), then set it as the wrapper in your **global**
  `~/.cargo/config.toml` — not a committed repo config, since a repo-level `rustc-wrapper`
  breaks contributors who don't have sccache installed:
  ```toml
  [build]
  rustc-wrapper = "sccache"
  ```
  For a cache **shared across machines / CI / ephemeral boxes**, point sccache at object
  storage (it supports S3, GCS, Azure, Redis, memcached) via `SCCACHE_*` env vars in your
  shell — read only when the sccache *server* starts, so `sccache --stop-server` after
  changing them. Mind the write mode: some backends default to read-only (e.g. GCS needs
  `SCCACHE_GCS_RW_MODE=READ_WRITE` or nothing is ever cached). Give the bucket a short
  delete-lifecycle so it self-cleans. Verify with `sccache --show-stats`. The win is on
  **dependency** crates; incremental debug builds of your own workspace are non-cacheable by design.

- **mold** is a linker 3-10x faster than GNU ld. Linking is the part sccache *can't* cache
  (a cache hit still relinks), so it's the biggest remaining win on incremental builds.
  Install (`sudo apt-get install -y mold`) and wire it **per-target** in `.cargo/config.toml`
  (repo or global) so it applies only to Linux:
  ```toml
  [target.x86_64-unknown-linux-gnu]
  rustflags = ["-C", "link-arg=-fuse-ld=mold"]
  ```
  Keep it Linux-scoped: mold can't link macOS Mach-O (its macOS port was discontinued), and
  Apple's default linker (ld-prime, Xcode 15+) is already fast — a *global* `-fuse-ld=mold`
  would break Mac builds. In Docker, `apt-get install mold` in the build stage and COPY the
  `.cargo/config.toml` in; pair with cargo-chef so dependency compilation is a cached layer.

AI coding agents — install the ones your team uses, each person authenticates their own:

```sh
curl -fsSL https://claude.ai/install.sh | bash    # Claude Code
npm install -g @openai/codex                      # Codex CLI (needs node from above)
```

Your cloud provider's CLI (gcloud / aws) is usually preinstalled on that cloud's images;
each person runs its auth login themselves.

Quality tooling — linters and checkers worth having on the box so agents and humans can run
the same checks everywhere. Preference: strict rule sets, fast compiled tools, cohesive
ecosystems. The per-language ones (Biome for TypeScript, ruff + ty for Python, Clippy with
pedantic lints for Rust) come in per-project via pnpm/uv/rustup — don't install those
globally. The cross-cutting ones are global:

```sh
sudo apt-get install -y shellcheck                # shell scripts
# Dockerfiles (applies ShellCheck rules to RUN lines; no apt package — grab the binary):
sudo curl -fsSL -o /usr/local/bin/hadolint \
  https://github.com/hadolint/hadolint/releases/latest/download/hadolint-Linux-x86_64 && \
  sudo chmod +x /usr/local/bin/hadolint
# Broken links in docs/markdown (Rust; via cargo, or grab a release binary):
cargo install --locked lychee
# Terraform/IaC (skip unless you manage infra from the box):
curl -fsSL https://get.opentofu.org/install-opentofu.sh | sh -s -- --install-method deb
curl -fsSL https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
```

## 7. Sync YOUR personal setup (each person)

On top of the shared baseline, your unix account is yours to furnish — push your own
settings up and authenticate tools with YOUR OWN credentials, never a teammate's:

```sh
scp ~/.gitconfig <NODE>:~/                    # then set your own git identity on the VM
scp ~/.zshrc <NODE>:~/                        # trim machine-specific lines after copying
# MCP servers your agent uses — user scope makes them available in every project:
ssh <NODE> 'claude mcp add --transport http --scope user <name> <mcp-url>'
```

Worth doing once: keep a `setup-devbox.sh` in your dotfiles repo with your personal
additions (aliases, extra CLIs, agent config) — the next devbox becomes one command.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `<NODE>` unreachable | Tailscale app not running or logged out on your laptop |
| ssh hangs on a login.tailscale.com URL | ACL `check` re-auth — open it; fix permanently via `accept` |
| mosh: "UDP port 60001 not reachable" | You're bypassing the tailnet (public IPs need open UDP; tailnet needs none) |
| mosh: "needs a UTF-8 native locale" | Your laptop's LANG (e.g. `en_SG.UTF-8`) isn't generated on the VM — cloud images only ship en_US. Fix: `ssh <NODE> 'sudo locale-gen <your LANG> && sudo update-locale'` |
| Copy/paste dead | **First: are you on mosh?** OSC 52 never traverses stock mosh — switch to an ssh tab or copy with terminal-native selection (Option+drag). On ssh: detach/reattach tmux, verify the `Ms=` line, then run the hop-isolation test below |
| App says "copied" inside tmux, clipboard empty (works outside tmux, over ssh) | Raw OSC 52 apps need `set -g set-clipboard on`; DCS-wrapping apps (AI agent CLIs) need `set -g allow-passthrough on` — set both |
| Pasting a screenshot/image does nothing | Expected on every transport: terminals paste text only. Ship the file and paste its path — `pshot` helper in section 5 |
| Trackpad scrolling cycles shell history in plain mosh | Expected: mosh uses the alternate screen (no scrollback), so terminals convert scroll to arrow keys there. Attach tmux (`tmux new -A -s main`) to scroll real history, or use an ssh tab for the terminal's native scrollback |
| Laggy typing | `tailscale ping <NODE>` — "via DERP" means relayed; direct paths usually return once both ends have real connectivity |
| Everything down (Tailscale outage) | GCP: `gcloud compute ssh <VM> --zone <ZONE> --tunnel-through-iap` · AWS: `aws ssm start-session --target <INSTANCE_ID>` |
| VM stopped | Start it via your cloud CLI/console — with a pinned IP nothing else changes |

### Clipboard hop-isolation test (run over ssh, not mosh)

When copy silently dies you have three suspects: tmux, the transport, or your terminal. Run
both of these inside tmux, then paste locally — whichever string arrives tells you which hop
works:

```sh
# full chain (tmux intercepts and re-emits via its Ms capability):
printf '\033]52;c;%s\007' "$(printf %s TMUX-PATH-OK | base64)"
# bypass tmux (raw passthrough straight to the transport/terminal):
printf '\033Ptmux;\033\033]52;c;%s\007\033\\' "$(printf %s DIRECT-PATH-OK | base64)"
```

Paste = `TMUX-PATH-OK` → everything works. Paste = `DIRECT-PATH-OK` only → tmux's clipboard
hop is broken (check the `Ms=` override and `set-clipboard`). Neither → the transport or
terminal is eating OSC 52 — if you're on mosh, that's expected (see the reality check above);
on ssh, check your terminal's clipboard-write permission. To test the terminal alone, run the
first printf in a plain local shell (no ssh, no tmux).

One trap from real debugging: when grepping captures for OSC 52, match the ESC byte
(`rg -a $'\x1b\]52'`) — matching a bare `]52` finds false positives in ordinary text.
