# Roadmap

Ideas parked for later. Nothing here is scheduled.

## macOS machine project

This repo provisions Linux (Fedora) machines only. A sibling project would do the same for a Mac, so moving
between machines feels the same.

- Same layering as here: the Mac project installs system things (Homebrew, casks, macOS defaults) and chezmoi applies
  the dotfiles. The dotfiles repo already supports macOS (`packages.yaml` has a `darwin` section).
- Linux equivalents already chosen for Mac apps: Rectangle → [gnometangle](https://github.com/aaronfeingold/gnometangle),
  Maccy → Clipboard History, Spotlight → GNOME's own search on Super+Space (`keyboard` role), Dock → Dash to Dock (`gnome_extensions` role).

## Same terminal on every machine, in bash and zsh

Goal: identical prompt, plugins and shortcuts whether the shell is bash or zsh, on Linux or macOS. Lives in the
dotfiles repo (chezmoi), since that already runs on both; this repo only installs packages and sets the login shell.

1. Shared core in `~/.config/shell/*.sh` (PATH, env, aliases, portable functions), sourced by both `.zshrc` and a
   chezmoi-managed `.bashrc`. Each rc file keeps only shell-specific bits (framework, completions, key bindings).
2. One prompt: Starship, configured once in `starship.toml`, instead of different oh-my-zsh / oh-my-bash themes.
3. Matching plugin sets:

   | Feature | zsh | bash |
   |---|---|---|
   | Framework | oh-my-zsh | oh-my-bash |
   | Suggestions as you type, syntax highlighting | zsh-autosuggestions, zsh-syntax-highlighting | ble.sh |
   | Tab menu | `menu_complete` | ble.sh menu |
   | History and file search | fzf | fzf |
   | Directory jumping | zoxide | zoxide |
   | git aliases | `git` plugin | `git` plugin |

4. Same packages on both OSes (starship, zoxide, fzf). macOS ships bash 3.2 and ble.sh needs 4.0+, so the Mac gets
   Homebrew bash and ble.sh only loads on bash 4+.
5. `user_shell` picks the login shell; the other shell is fully configured too.
6. Tests: the container tests start `zsh -i` and `bash -i` and check the prompt, plugins and aliases load; a macOS
   CI job in the dotfiles repo runs `chezmoi apply` with the same checks.
