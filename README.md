# Fedora dev machine

Ansible that turns a fresh **Fedora 44** install into a modern dev machine: a laptop, a VM, or a
cloud instance you build on. It handles the system (packages, Docker, databases, cloud CLIs, NVIDIA/CUDA
when there's a GPU); your personal config (shell, tmux, editor, git) comes from **your own dotfiles
repo**, applied with [chezmoi](https://chezmoi.io) as the last step.

Want a ready-made image instead of running this on every new instance? See
[fedora-dev-images](https://github.com/aaronfeingold/fedora-dev-images) (Packer, AWS + GCP).

Nothing personal is stored in this repo. Your name, email and dotfiles repo come from env vars,
a gitignored `local.yml`, or your `git config`.

## 1. Configure

Pick one:

```sh
# a) env vars
export GIT_NAME="Your Name" GIT_EMAIL="you@example.com" DOTFILES_REPO="you/dotfiles"

# b) local file (gitignored, loaded automatically by run_playbook)
cp local.example.yml local.yml && $EDITOR local.yml
```

| Setting | Env var | Default |
|---|---|---|
| `git_name`, `git_email` | `GIT_NAME`, `GIT_EMAIL` | `git config --global user.name/email` |
| `dotfiles_repo` | `DOTFILES_REPO` | empty = skip dotfiles. `user/repo` (GitHub) or any git URL |
| `dotfiles_ssh` | `DOTFILES_SSH` | `true`: clone `user/repo` over SSH (needed for private repos) |
| `user_shell` | `USER_SHELL` | `/usr/bin/zsh` |
| `install_databases`, `install_gui`, `install_ml`, `install_cloud_clis` | (use `-e` or `local.yml`) | see `group_vars/` |

No dotfiles yet? Start from the template [dotfiles-kit](https://github.com/aaronfeingold/dotfiles-kit);
repos made from it work here with no extra setup.

Precedence (highest first): `-e` on the command line, `local.yml`, env vars, `git config`.
For `-e` values with spaces use JSON: `-e '{"git_name": "Ada Lovelace"}'`.

## 2a. Run it on the machine itself

```sh
curl -fsSL https://raw.githubusercontent.com/aaronfeingold/fedora-dev-machine/main/bootstrap | bash
cd ~/Code/fedora-dev-machine
./run_playbook
```

`bootstrap` installs `ansible-core` and clones this repo (set `ANSIBLE_REPO_URL` to use your fork).
`run_playbook` targets this machine with the `workstation` settings (databases + GUI apps on) and
asks for your sudo password only if needed. Everything after it is passed to `ansible-playbook`:

```sh
./run_playbook --tags dotfiles          # just one role
./run_playbook --check --diff           # dry run
./run_playbook -e install_ml=true       # one-off override
```

## 2b. Run it on a remote machine (from your Mac or another Linux box)

```sh
brew install ansible        # macOS; on Fedora: sudo dnf install ansible-core
ssh-add                     # your GitHub key, so the remote can clone a private dotfiles repo

./run_playbook -i 203.0.113.10, -u fedora                                # one host (note the comma)
./run_playbook -i 203.0.113.10, -u fedora -e @group_vars/gpu_cloud.yml   # with cloud GPU settings
```

Or list hosts under `[gpu_cloud]` in `inventory.ini` and run `./run_playbook -l gpu_cloud`.
Your laptop's `git config` / env vars / `local.yml` are used, and SSH agent forwarding lets the remote
clone your dotfiles without copying any keys to it.

## 3. Test it on a Mac (or any machine with Docker/Podman)

```sh
tests/run                                         # workstation settings (databases + GUI apps)
tests/run -e install_ml=true -e git_name=Test -e git_email=t@example.com
FEDORA_VERSION=43 tests/run
KEEP=1 tests/run && docker exec -it -u dev fedora-ansible-test zsh    # poke around afterwards
```

This boots a throwaway Fedora container **with systemd running**, runs the playbook twice as a normal
sudo user (the second run must report `changed=0`), then checks Docker, PostgreSQL and Valkey actually work.
`local.yml` is not copied in; pass test values with `-e`.

It can't test GPUs, kernel drivers or reboots. For those use a real VM (UTM/Parallels/QEMU with the
Fedora 44 ISO, then section 2b against its IP) or a cloud instance.

## Roles

| Role | Tags | Runs when | What |
|---|---|---|---|
| base | base | always | `dnf upgrade`, core CLI tools (git, zsh, tmux, neovim, ripgrep, fzf, lazygit, gh…), zsh as login shell |
| python | base, python | always | uv, pipx, pyenv (+ build deps), poetry |
| node | base, node | always | nvm + Node LTS |
| docker | base, docker | always | Docker CE + compose/buildx, user in `docker` group |
| databases | db | `install_databases` | PostgreSQL + Valkey (Redis-compatible), a DB role for your user |
| cloud_cli | cloud | `install_cloud_clis` | AWS CLI v2 (PGP-verified), gcloud, packer |
| workstation | gui | `install_gui` | VS Code, Ghostty, JetBrainsMono Nerd Font |
| nvidia | gpu | NVIDIA GPU detected (or `gpu_force`) | open driver, CUDA toolkit, nvidia-container-toolkit |
| ml | ml | `install_ml` | uv venvs `~/.venvs/torch` and `~/.venvs/tf` (CUDA wheels with a GPU, CPU wheels without) |
| dotfiles | dotfiles | `dotfiles_repo` set | chezmoi init + apply (first run only; then use `chezmoi update`) |

Machine types live in `group_vars/`: `all.yml` (defaults), `workstation.yml` (laptop: databases + GUI),
`gpu_cloud.yml` (ML on).

## Notes

- **Fedora upgrades:** repos use `$releasever`, so after `dnf system-upgrade` just re-run the playbook.
  NVIDIA's CUDA repo can lag a new Fedora release by a few weeks.
- **NVIDIA:** driver and CUDA both come from NVIDIA's repo so versions match; the role stops if RPM Fusion's
  `akmod-nvidia` is installed. Open kernel modules need a Turing-or-newer GPU (T4, A10G, L4, A100, H100, RTX 20xx+).
- **Secure Boot:** DKMS signs modules with its own key. On bare metal either disable Secure Boot or enroll the key
  once: `sudo mokutil --import /var/lib/dkms/mok.pub`, reboot, accept it in the blue MOK screen.
