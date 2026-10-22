# AJF Fedora machine

Ansible for turning a fresh **Fedora 44** box into my dev machine: the Framework laptop, a VM,
or a cloud GPU instance for training. System stuff lives here; user config (zsh, tmux, nvim, git,
VS Code, Claude Code) comes from my [dotfiles](https://github.com/aaronfeingold/dot-files) via chezmoi,
which the last role applies.

## Quick start (this machine)

```sh
curl -fsSL https://raw.githubusercontent.com/aaronfeingold/ajf-fedora-workstation-ansible/main/bootstrap | bash
cd ~/Code/ajf-fedora-workstation-ansible && ./run_playbook
```

`run_playbook` passes everything to `ansible-playbook` and asks for your sudo password only when needed.

## Cloud GPU box

From your laptop (agent forwarding lets the box clone the private dotfiles with your key):

```sh
ssh-add                                   # make sure your GitHub key is in the agent
./run_playbook -i 203.0.113.10, -u fedora -e @group_vars/gpu_cloud.yml
# or add it to [gpu_cloud] in inventory.ini, then:
./run_playbook -l gpu_cloud
```

On an NVIDIA machine the `nvidia` role installs the driver, CUDA toolkit and container toolkit,
reboots if the driver isn't loaded yet, and checks `nvidia-smi`. The `ml` role then builds
`~/.venvs/torch` and `~/.venvs/tf` with CUDA wheels and prints whether each framework sees the GPU.

## Roles

| Role | Tags | Runs on | What |
|---|---|---|---|
| base | base | all | `dnf upgrade`, core CLI tools, zsh as login shell |
| python | base, python | all | uv, pipx, pyenv (+ build deps), poetry |
| node | base, node | all | nvm + Node LTS |
| docker | base, docker | all | Docker CE + compose/buildx, user in `docker` group |
| databases | db | `install_databases` | PostgreSQL + Valkey (Redis-compatible), role for your user |
| cloud_cli | cloud | `install_cloud_clis` | AWS CLI v2 (PGP-verified), gcloud, packer |
| workstation | gui | `install_gui` | VS Code, Ghostty, JetBrainsMono Nerd Font |
| nvidia | gpu | NVIDIA GPU detected (or `gpu_force`) | open driver, CUDA toolkit, nvidia-container-toolkit |
| ml | ml | `install_ml` | uv venvs: PyTorch, TensorFlow, transformers, lightning, jupyter… (CPU wheels if no GPU) |
| dotfiles | dotfiles | all | chezmoi init + apply (first run only) |

Toggles are in `group_vars/`: `all.yml` (defaults), `workstation.yml` (Framework: databases + GUI on),
`gpu_cloud.yml` (ML on). Override any of them with `-e`, e.g. `./run_playbook -e install_ml=true`.

Run a subset: `./run_playbook --tags gpu,ml`. Dry run: `./run_playbook --check --diff`.

## Notes

- **Fedora version upgrades:** repos use `$releasever` / `ansible_distribution_major_version`, so after
  `dnf system-upgrade` just re-run the playbook. NVIDIA's CUDA repo usually lags new Fedora releases by a
  few weeks; check `https://developer.download.nvidia.com/compute/cuda/repos/` for `fedora<N>`.
- **NVIDIA driver source:** NVIDIA's repo (not RPM Fusion), so driver and CUDA versions match. The role
  refuses to run if RPM Fusion's `akmod-nvidia` is installed.
- **Secure Boot:** DKMS signs modules with its own key, which the firmware doesn't trust yet. Cloud
  instances usually have Secure Boot off; on bare metal either disable it or enroll DKMS's key once:
  `sudo mokutil --import /var/lib/dkms/mok.pub`, reboot, and accept it in the blue MOK screen.
- **Open kernel modules** need a Turing-or-newer GPU (T4, A10G, L4, A100, H100, RTX 20xx+).
- The Framework has no NVIDIA GPU, so `gpu` is skipped there automatically.
