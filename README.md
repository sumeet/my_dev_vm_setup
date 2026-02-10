# Forkable Dev Boxes with LXD + ZFS + One Zellij

## What you get

* **A “base” container** you set up once (toolchain, dotfiles, etc).
* **Instant fork/clones** of that base using **ZFS copy-on-write** (cheap snapshots/clones; disk grows only with changes).
* **One Zellij on the host**, where **each Zellij session name == one container**.
* New tabs/panes automatically open **inside that container**, not on the host.
* New tabs/panes also open in the **same working directory** as your last tab.

---

## Transcript

Excerpt of the original ChatGPT thread that led to this setup: `docs/transcript.md`.

---

## Why ZFS matters (copy-on-write)

LXD’s default “dir” storage does *not* give you cheap forks. ZFS does.

With ZFS as the LXD storage backend:

* `lxc snapshot base …` is cheap.
* `lxc copy base@… dev1` is near-instant and **copy-on-write**.
* Clones don’t consume full disk up front; they only store deltas.

That CoW behavior is the whole reason this scales.

---

# Part 1 — LXD + ZFS (host)

### 1) Install LXD and ZFS pool

(Host is Ubuntu-ish)

```bash
sudo snap install lxd
sudo lxd init --minimal

sudo apt update
sudo apt install -y zfsutils-linux
sudo lxc storage create zpool1 zfs size=100GiB
sudo zfs set compression=zstd-3 zpool1
sudo zfs set atime=off zpool1
sudo zfs set dedup=off zpool1

# Make new containers use zpool1 by default
lxc profile device set default root pool zpool1
```

### 2) Move your existing containers (if any) onto ZFS

```bash
lxc stop base || true
lxc move base -s zpool1
```

### 3) Snapshot base (recommended)

```bash
lxc snapshot base golden
```

---

# Part 2 — Zellij single-host “sessions = containers”

## Core idea

Zellij can be configured with a **default shell**. We set the default shell to a wrapper script (`ctrsh`).

Every time Zellij spawns a shell (new pane, new tab), it runs `ctrsh`.

`ctrsh` reads `$ZELLIJ_SESSION_NAME`:

* if session is `dev1`, it `lxc exec`s into container `dev1`
* if the container doesn’t exist, it clones it from `base` (or `base/golden`), starts it, and then enters it

So a “session” becomes a persistent container workspace.

---

## 1) Host: `ctrsh` wrapper

Put this somewhere like `~/bin/ctrsh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
ctr="${ZELLIJ_SESSION_NAME:-}"

# host session or no session -> host shell
if [[ -z "${ctr}" || "${ctr}" == "host" ]]; then
  exec "${SHELL:-bash}" -l
fi

# create container on-demand (recommended: copy snapshot if you have one)
if ! lxc info "$ctr" >/dev/null 2>&1; then
  if lxc info base/golden >/dev/null 2>&1; then
    lxc copy base/golden "$ctr"
  else
    lxc copy base "$ctr"
  fi
  # optional: per-container disk quota so one box can't eat the pool
  lxc config device set "$ctr" root size 30GiB || true
fi

# ensure running
if ! lxc info "$ctr" | grep -qi 'Status: *running'; then
  lxc start "$ctr" >/dev/null 2>&1 || true
fi

# fetch last PWD recorded by the user's shell inside the container
cwd="$(lxc exec "$ctr" -- sudo -u dev bash -lc '
  f="$HOME/.zellij_pwd"
  d="$HOME"
  if [ -r "$f" ]; then read -r d <"$f"; fi
  [ -d "$d" ] || d="$HOME"
  printf %s "$d"
')"

# enter container in that directory; avoid login shell so CWD sticks
# change "bash" to "fish" or "zsh" if you want.
exec lxc exec -t "$ctr" --cwd "$cwd" -- sudo -u dev bash
```

Make it executable:

```bash
chmod +x ~/bin/ctrsh
```

> Note: This assumes a `dev` user exists inside the container and `sudo` is available. (You usually set that up once in `base`.)

---

## 2) Host: tell Zellij to use the wrapper

In `~/.config/zellij/config.kdl`, set:

```kdl
default_shell "/home/YOURUSER/bin/ctrsh"
```

Restart Zellij after changing this.

---

## 3) Usage

* Create/attach a “dev box”:

  ```bash
  zellij attach -c dev1
  ```

  This creates/starts container `dev1` and drops you into it.

* New tab/pane:

  * It still lands inside `dev1` because Zellij always spawns via `ctrsh`.

* Switch boxes:

  ```bash
  zellij attach -c dev2
  ```

---

# Part 3 — Working directory tracking (works with fish, bash, zsh)

## Goal

When you `cd` in one pane, new tabs/panes should open in that same directory.

The mechanism:

* Inside the container, your shell writes the current directory to `~/.zellij_pwd`.
* Host wrapper reads it and uses `lxc exec --cwd`.

## Fish version (your snippet)

Create: `~/.config/fish/conf.d/zellij-pwd.fish`

```fish
function __zellij_pwd_sync --on-variable PWD
    echo $PWD > ~/.zellij_pwd
end
echo $PWD > ~/.zellij_pwd
```

## Bash version

Add to `~/.bashrc` inside the container:

```bash
__zellij_pwd_sync() { printf '%s\n' "$PWD" > "$HOME/.zellij_pwd"; }
PROMPT_COMMAND="__zellij_pwd_sync${PROMPT_COMMAND:+; $PROMPT_COMMAND}"
__zellij_pwd_sync
```

## Zsh version

Add to `~/.zshrc` inside the container:

```zsh
__zellij_pwd_sync() { print -r -- "$PWD" > "$HOME/.zellij_pwd" }
autoload -Uz add-zsh-hook
add-zsh-hook chpwd __zellij_pwd_sync
__zellij_pwd_sync
```

Pick whichever shell you use—same file, same effect.

---

# Part 4 — “base” container requirements

Inside `base`, you generally want:

* a normal user (e.g. `dev`)
* `sudo`
* your preferred shells/editors/toolchain

Example (rough) inside base:

```bash
# create user + sudo
useradd -m -s /bin/bash dev
usermod -aG sudo dev
apt install -y sudo
echo 'dev ALL=(ALL) NOPASSWD:ALL' >/etc/sudoers.d/90-dev
chmod 440 /etc/sudoers.d/90-dev
```

Then snapshot it:

```bash
lxc snapshot base golden
```

---

# Mental model recap

* **ZFS gives you CoW forks** so cloning dev environments is cheap.
* **Zellij sessions are names**, and you use those names as container names.
* Zellij spawns shells via `ctrsh`.
* `ctrsh` “teleports” every new pane/tab into the right container and directory.

That’s the whole trick.
