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
| [`devbox-setup`](devbox-setup/SKILL.md) | Provision + harden a team devbox (no public SSH; IAP/SSM break-glass), onboard teammates via Tailscale SSH, and get mosh + tmux sessions with working clipboard that survive everything. |

## The stack

- **Tailscale + Tailscale SSH** — the VM has no internet-facing ports; your tailnet login is your credential
- **mosh** — connections survive lid-close, network switches, and IP changes
- **tmux + resurrect/continuum** — sessions survive VM reboots, with a clipboard fix that makes
  copy/paste actually work through mosh
- **Cloud-native break-glass** — GCP IAP tunnel or AWS SSM Session Manager for when the tailnet is down

## License

MIT
