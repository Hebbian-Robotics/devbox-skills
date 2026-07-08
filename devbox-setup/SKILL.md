---
name: devbox-setup
description: Set up a shared cloud dev VM ("devbox") with zero public ports — Tailscale + Tailscale SSH for identity-based auth, mosh + tmux for sessions that survive laptop sleep and reboots, working clipboard through mosh, and a cloud-native break-glass path (GCP IAP or AWS SSM). Use when provisioning a team devbox, onboarding a teammate onto one, or debugging devbox connection/clipboard/session problems.
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

### Tailscale on the VM (all clouds)

```sh
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname <NODE>      # prints a login URL — approve it in a browser
sudo tailscale set --ssh=true            # identity-based SSH; add --accept-risk=lose-ssh if connected via tailnet
```

No inbound firewall rule is needed for Tailscale — WireGuard punches out on UDP and falls back
to TCP-443 relays, so it works even from UDP-hostile networks (mosh's UDP rides inside the tunnel).

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

`~/.tmux.conf` — the `Ms` line is load-bearing: mosh only forwards OSC 52 clipboard writes that
use an explicit `c` target and BEL terminator, so without it, copying from tmux silently dies:

```
set -g mouse on
set -g history-limit 100000
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:Tc"
setw -g mode-keys vi
set -g escape-time 10
set -g renumber-windows on
# Clipboard through mosh (explicit "c" target + BEL, or mosh drops the copy):
set -as terminal-overrides ',xterm*:Ms=\E]52;c;%p2%s\007'
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
mosh <NODE> -- tmux new -A -s main
```

- Survives lid-close, Wi-Fi switches, IP changes — the session resumes instantly on wake.
- `ctrl-b d` detaches; everything keeps running server-side.
- **Copy**: mouse-select inside tmux → lands on your local clipboard. Hold **Option** while
  dragging to bypass tmux entirely (native terminal selection). The clipboard fix only applies
  to clients attached after the config loaded — if copy is dead, detach/reattach once.
- **Scrollback**: `ctrl-b [` (mosh has no native scrollback; tmux's 100k-line history is it).

## 6. Sync YOUR tooling (each person)

Your unix account on the VM is yours to furnish — push your own settings up:

```sh
scp ~/.gitconfig <NODE>:~/                    # then set your own git identity on the VM
scp ~/.zshrc <NODE>:~/                        # trim machine-specific lines after copying
# CLIs you use daily (auth each with YOUR OWN credentials, never a teammate's):
ssh <NODE> 'curl -fsSL https://claude.ai/install.sh | bash'
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `<NODE>` unreachable | Tailscale app not running or logged out on your laptop |
| ssh hangs on a login.tailscale.com URL | ACL `check` re-auth — open it; fix permanently via `accept` |
| mosh: "UDP port 60001 not reachable" | You're bypassing the tailnet (public IPs need open UDP; tailnet needs none) |
| Copy/paste dead | Detach/reattach tmux; verify the `Ms=` line exists in `~/.tmux.conf` |
| Everything down (Tailscale outage) | GCP: `gcloud compute ssh <VM> --zone <ZONE> --tunnel-through-iap` · AWS: `aws ssm start-session --target <INSTANCE_ID>` |
| VM stopped | Start it via your cloud CLI/console — with a pinned IP nothing else changes |
