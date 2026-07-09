# devbox-skills

[Agent skills](https://skills.sh/) for setting up shared cloud dev VMs ("devboxes") the sane way:
**zero public ports, identity-based auth, and sessions that survive laptop sleep, Wi-Fi roaming,
and VM reboots.** Cloud-agnostic, with GCP and AWS provisioning + break-glass variants included.

Maintained by [Hebbian Robotics](https://hebbianrobotics.com).

## Install

```sh
npx skills add Hebbian-Robotics/devbox-skills
```

## Skills

| Skill | What it does |
|---|---|
| [`devbox-setup`](devbox-setup/SKILL.md) | Provision + harden a team devbox (no public SSH; IAP/SSM break-glass), onboard teammates via Tailscale SSH, resilient ssh/mosh + tmux sessions with honest clipboard guidance, and a recommended developer toolchain (uv, fnm/pnpm, gh, ripgrep, AI coding agents). |

## The stack

- **Tailscale + Tailscale SSH** — the VM has no internet-facing ports; your tailnet login is your credential
- **ssh over the tailnet** for daily work (stable path, full clipboard support), **mosh** for
  roaming/bad networks — with an honest account of what each transport can and can't do
- **tmux + resurrect/continuum** — sessions survive VM reboots, with the three clipboard
  settings that make copy/paste work for both copy-mode and agent CLIs
- **A developer toolchain baseline** — uv, fnm + corepack/pnpm, rustup, gh, ripgrep/fd, and
  AI coding agents (Claude Code, Codex), installed the non-stale way
- **Cloud-native break-glass** — GCP IAP tunnel or AWS SSM Session Manager for when the tailnet is down

## License

MIT
