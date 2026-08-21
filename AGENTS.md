# AGENTS.md — Chezmoi Dotfiles (@josacar)

## What This Is

A [chezmoi](https://chezmoi.io) dotfiles repository managed by **Jose Luis Salas** (`josacar`), converted from an earlier `freshshell/fresh` setup. It manages shell, editor, and tool configuration across **macOS** and **Linux** (Debian-based) machines, with a **personal/work profile** toggle.

---

## Essential Commands

| Action | Command |
|---|---|
| Apply dotfiles | `chezmoi apply` |
| Diff changes | `chezmoi diff` |
| Add file to repo | `chezmoi add ~/.somefile` |
| Edit tracked file | `chezmoi edit ~/.somefile` |
| Init new machine | `sh -c "$(curl -fsLS https://chezmoi.io/get)" -- init --apply josacar` |
| Test apply (dry-run) | `chezmoi apply --dry-run --verbose` |

> No Makefile, scripts/, or CI config exists. The repo is pure chezmoi source of truth.

---

## Directory/File Naming Conventions

chezmoi uses **prefix-based** mapping for source filenames:

| Prefix | Meaning | Example → Target |
|---|---|---|
| `dot_` | Leading `.` | `dot_bashrc` → `~/.bashrc` |
| `private_` | `0600` permissions | `private_dot_ssh/private_config` → `~/.ssh/config` |
| `executable_` | `+x` permission | `dot_local/private_bin/executable_turbo` → `~/.local/bin/turbo` |
| `readonly_` | `0400` (or `0444`) permission | `dot_config/readonly_starship.toml` → `~/.config/starship.toml` |
| `.tmpl` suffix | Go template — processed before apply | `dot_bashrc.tmpl` → `~/.bashrc` (processed) |
| `run_onchange_after_apply_*` | Script run every time after `chezmoi apply` detects changes | `run_onchange_after_apply_1_install-packages.sh.tmpl` |

**Key detail**: Prefixes compose (e.g., `dot_local/private_bin/executable_turbo`). Nested directory structure is preserved — `dot_config/nvim/init.lua` lands at `~/.config/nvim/init.lua`.

---

## Template System

Chezmoi uses Go templates with custom functions. All `.tmpl` files get processed.

### Template Functions in Use

| Function | Purpose |
|---|---|
| `promptBool` / `promptString` | Interactive prompts during `chezmoi init` — defines personal vs work profile |
| `include "path"` | Include a partial file **without** further template processing |
| `includeTemplate "path" .` | Include a partial file **with** further template processing |
| `hasKey . "key"` | Check if a key exists in template data |
| `eq .chezmoi.os "darwin"` | Conditional on OS (darwin/linux) |
| `sha256sum` | Compute SHA256 (used in comments to track content changes) |
| `gitHubLatestReleaseAssetURL` | Fetch latest release URL from GitHub (chezmoi built-in) |
| `.chezmoi.arch` | System architecture (amd64, arm64) |

### Partial System

Reusable chunks live in `partials/bashrc/` with `.sh` extension (some `.tmpl`, some not). The main `dot_bashrc.tmpl` assembles bashrc from partials in a specific order:

```
bash_early → common → custom → brew (macOS only) → zerobrew (macOS only) → git → tools → ruby → completions → asdf → assume → go → iterm → mise → ble.sh → z.sh
```

### Template Data Flow

1. `.chezmoi.toml.tmpl` prompts user during `chezmoi init` for `work` (bool), `email`, `signingkey` — these become `.work`, `.email`, `.signingkey`
2. `.chezmoidata.toml` defines `.tools.bw.version` and `.tools.bws.version` (referenced by `.chezmoiexternal.toml.tmpl`)
3. `.chezmoiignore.tmpl` controls which files to skip per OS and work profile
4. `.chezmoiexternal.toml.tmpl` manages external downloads (binaries, fonts, z.sh, work dotfiles)

### External Assets Downloaded

Managed via `.chezmoiexternal.toml.tmpl`:
- **Scripts**: `z.sh` (rupa/z)
- **Fonts**: CodeNewRoman Nerd Font, Meslo Nerd Font (OS-conditional paths)
- **Binaries**: age, age-keygen, gdu, mitamae, bw (Bitwarden), bws (Bitwarden Secrets), usage (jdx/usage), gh (GitHub CLI)
- **Git repos**: work profile dotfiles (gitlab.com/josacar/work-dotfiles)

---

## Cross-Platform Architecture

### macOS (darwin)
- **Package mgr**: Homebrew (`Brewfile.tmpl` → Brewfile, installed via `brew bundle`)
- **Shell**: bash from brew, Homebrew paths
- **Terminal**: wezterm (WebGPU frontend)
- **macOS defaults**: Set in `run_onchange_after_apply_1_install-packages.sh.tmpl` (dock, finder, safari, keyboard, trackpad, screenshots, etc.)
- **Rancher Desktop**: Docker host path is `unix:///Users/joseluis/.rd/docker.sock`

### Linux (Debian-based)
- **Package mgr**: apt (`apt-get install ripgrep vim neovim gpg sudo wget curl fd-find fzf`)
- **Shell**: system bash, system paths
- **ble.sh**: loaded from `/usr/share/blesh/ble.sh`
- **No Homebrew/Brewfile** — Brewfile and Library/ are gitignored on Linux

### Work vs Personal Profile

The `.work` key (set during `chezmoi init`) controls:
- **work=true**: Includes SSH config from `~/.dotfiles/work/.ssh_config`, installs Granted (AWS), installs SelfServeManifest for Mac managed installs, shows install instructions. Excludes opencode and crush configs.
- **work=false** (or unset): Excludes work SSH config, installs personal brew casks (firefox, docker, slack, spotify, etc.). Excludes `.ssh/curro`.

---

## Shell Environment

### bash Configuration
- **vi-mode** (`set -o vi` in `partials/bashrc/custom.sh`)
- **Prompt**: Custom `PS1` (green user:blue cwd, git branch in red via `__git_ps1`), not starship (starship is installed but shell prompt overrides it)
- **History**: `HISTCONTROL=ignoreboth`, `HISTSIZE=1000`, `HISTFILESIZE=2000`
- **LC_ALL**: `en_US.UTF-8`

### Git Aliases (in `partials/bashrc/git.sh`)

Extensive git aliases (`ga`, `gc`, `gd`, `gs`, `gp`, `grom`, etc.) and helper functions (`git_current_branch`, `git_default_branch`, `glp`, `gcf`, `gpr`). Too many to list — most start with `g` + letter(s). See the file for the full list.

### Tool Management
- **mise** (replaces asdf) — activated in bash via `eval "$(mise activate bash)"`
- **asdf** — still available on macOS, legacy support path
- **zerobrew** — macOS-only alternative package manager
- **fzf** — activated via `eval "$(fzf --bash)"`

### Key Aliases (non-git)
- `vim` → `nvim`
- `ls` → `eza` (macOS-aware)
- `ack` → `ack-grep` (if available)
- `b` → `bundle`, `be` → `bundle exec`, `ber` → `bundle exec rspec`

---

## Editor Configuration

### Neovim (primary)

**Framework**: [LazyVim](https://www.lazyvim.org/) (v8, installed via lazy.nvim)

Key config:
- `init.lua`: Bootstrap lazy.nvim, custom treesitter parser for Crystal (from `~/code/tree-sitter-crystal`)
- `lua/config/options.lua`: Leader `,`, localleader `\`, snacks animations disabled, Ruby formatter set to `standardrb`
- `lua/config/keymaps.lua`: `\\` → toggle comment (gcc)
- `lua/config/colorscheme.lua`: Tokyo Night (`folke/tokyonight.nvim`)
- `lua/config/autocmds.lua`: Empty (just comments)
- `lua/plugins/init.lua`: Treesitter parsers (bash, hcl, html, js, json, lua, markdown, python, terraform, tsx, typescript, vim, yaml), codecompanion.nvim (Gemini adapter)
- `lua/plugins/example.lua`: Example plugin spec (disabled by default)
- `lazyvim.json`: Extras enabled — `editor.fzf`, `lang.ruby`, `lang.terraform`
- `.neoconf.json`: neodev library plugins enabled, lua_ls via neoconf
- `stylua.toml`: 2-space indent, 120 column width

Formatting: Stylua with 2 spaces, 120 column width.

### Vim (legacy)

Still configured in `dot_vim/` with vim-plug plugin manager. Plugin list in `vimrc/plug-bundle.vim` with pinned versions in `plug-bundle-versions.vim`. Automatically updated via `run_onchange_after_apply_2_vim-update-plugins.sh.tmpl`. Uses the `~/.vim-bundle/` directory for plugins.

### Tmux
- Prefix: `C-a` (not `C-b`)
- Reload: `r`
- vi-style copy mode
- Vim-like pane movement: `h/j/k/l`
- Split: `s` (vertical), `v` (horizontal)
- Status bar shows git branch in current pane dir
- `dot_tmux.conf` → `~/.tmux.conf`

### Wezterm
- WebGPU frontend (`config.front_end = "WebGpu"`)
- Integrated buttons + resize window decorations
- JetBrains Mono font, 16pt
- iTerm2 color scheme

---

## Security / Private Data

- **SSH config**: Template in `private_dot_ssh/private_config.tmpl` — conditionally includes work config. Host entries for personal machines (yojimbo, gigalmesh, openwrt, rock-3a).
- **GPG config**: `private_dot_gnupg/gpg-agent.conf.tmpl` (arch-specific socket paths for macOS/Linux)
- **Git signing**: GPG-signs all commits (`commit.gpgsign = true`)
- **Git email/GPG key**: Prompted during `chezmoi init`, defaults to `josacar@users.noreply.github.com` and key `3B5BB81B203269D79D8492D9BE7CDF9012256B23`

---

## Git Config (dot_gitconfig.tmpl)

Templated with `.name`, `.email`, `.signingkey` from init prompts. Key settings:
- `git rebase` autosetup, `pull.rebase = true`, `rebase.autoStash = true`
- `push.default = current`
- `core.autocrlf = input`
- `core.pager = less -FRSX`
- Git aliases (many)
- `difftool = vimdiff`, `mergetool = vimdiff`
- **Per-host identity via `includeIf "hasconfig:remote.*.url:..."`**: global `[user]` (from init prompts) is the fallback; four `includeIf` rules in `identity-github` / `identity-gitlab` override it for repos whose remote matches. Github remote → `josacar@users.noreply.github.com`; Gitlab remote → `1990502-josacar@users.noreply.gitlab.com`. Patterns must anchor both the `user@host:` scheme and the path: ssh is `git@github.com:*/**`, https is `https://github.com/**` (git's wildmatch `**` only crosses `/` when preceded by `/` or at pattern start — a bare `*github.com*` will NOT match).

---

## Important Gotchas

1. **`.gitignore`** is tracked as `.gitignore` (not `dot_gitignore`) — chezmoi doesn't manage `~/.gitignore`, this is the repo's own gitignore (currently only ignores `.mise.toml`).
2. **`partials/` is gitignored** at the top level via `.chezmoiignore.tmpl` (excluded because partials are not target files — they're included inline by `dot_bashrc.tmpl`).
3. **`README.md` is gitignored** via `.chezmoiignore.tmpl` — it's not deployed as a dotfile, it's repo documentation.
4. **`dot_vim/.chezmoiexternal.toml.tmpl`** is a secondary external config — check it if you add/need vim plugin external dependencies.
5. **Ruby install**: Uses `RUBY_CONFIGURE_OPTS="--with-jemalloc --enable-yjit"` via mise — OS-specific paths for jemalloc.
6. **Ripgrep config** (`dot_ripgreprc`) overrides with `--no-ignore` — ripgrep ignores `.gitignore` by default in this setup.
7. **Starship prompt** has most modules disabled — the actual prompt is a custom `PS1` with `__git_ps1`, starship is just used for a few remaining elements.
8. **Docker host** is set to Rancher Desktop's socket on macOS; on Linux it uses the default socket.
9. **Shell completion files** (`completions/git.sh`, `completions/tmux.sh`) are full scripts sourced into bashrc, not separate completion files.
10. **Bitwarden CLI versions** are pinned in `.chezmoidata.toml` — update there rather than in the external URLs.
11. **Opencode and Crush configs** (`.config/opencode/`, `.config/crush/`) are excluded on work profiles via `.chezmoiignore.tmpl`.

---

## Key Files Reference

| File | Purpose |
|---|---|
| `.chezmoi.toml.tmpl` | Main config, prompts for work profile |
| `.chezmoiexternal.toml.tmpl` | External binary/script downloads |
| `.chezmoiignore.tmpl` | Per-OS, per-profile ignore rules |
| `.chezmoidata.toml` | Tool version pins (bw, bws) |
| `dot_bashrc.tmpl` | bashrc assembly from partials |
| `partials/bashrc/custom.sh` | PS1, env vars, aliases, fixssh |
| `partials/bashrc/git.sh` | All git aliases and helpers |
| `partials/bashrc/ruby.sh` | Rails/bundle/rspec aliases |
| `Brewfile.tmpl` | macOS Homebrew packages (conditional on work profile) |
| `run_onchange_after_apply_1_install-packages.sh.tmpl` | macOS defaults + brew bundle + apt packages |
| `run_onchange_after_apply_2_vim-update-plugins.sh.tmpl` | Vim plugin update script |
| `dot_gitconfig.tmpl` | Git config (templated with name/email/signingkey) |
| `dot_config/nvim/init.lua` | Neovim entrypoint |
| `dot_config/nvim/lazyvim.json` | LazyVim version, extras config |
| `dot_config/crush/crush.json` | Crush AI assistant config (local openai-compat provider) |
