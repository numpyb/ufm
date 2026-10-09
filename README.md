# UFW Manager

A cross-platform desktop application for managing [UFW](https://launchpad.net/ufw) (Uncomplicated Firewall) firewalls — locally or on remote servers over SSH. It gives you the everyday UFW operations (enable/disable, defaults, logging, rules, app profiles) in a point-and-click UI, plus structured import/export of an entire firewall configuration.

Built in Rust with [eframe/egui](https://github.com/emilk/egui) (GUI) and [russh](https://github.com/Eugeny/russh) (pure-Rust SSH). No GTK, Qt, or libssh dependency — the binary links only libc and talks to the display server via the standard GL/EGL and X11/Wayland libraries already installed on a normal desktop.

---

## Features

- **Local and remote management** — manage the firewall on this machine, or add any number of servers and manage them over SSH from the same window (switch endpoints with one dropdown).
- **Overview** — firewall status, incoming/outgoing/routed default policies, logging level, enable / disable / reload, and the raw `ufw status verbose` output.
- **Rules** — full-text filterable table of every rule (action, direction, To, From, interface, comment), add rules with a guided builder and live command preview, dry-run mode, and delete with confirmation.
- **Applications** — browse installed UFW application profiles (`ufw app list`), inspect their details (`ufw app info`), and allow/deny them in either direction.
- **Servers** — a saved address book of SSH endpoints (key or password auth, optional custom key path, optional sudo), with per-server connect state.
- **Import / export** — export the whole firewall to a human-readable JSON bundle or a runnable shell script; import either, in **merge** or **replace** mode.
- **Command log** — every command run, with its output, so you can see exactly what happened.
- **Dark / light theme** (remembered between launches), non-blocking background jobs with progress indicator, and automatic recovery prompts for missing sudo / SSH passwords and key passphrases.

## Requirements

### Runtime

- Linux, macOS, or Windows desktop.
- **UFW installed on the machine you manage** (local or remote). The app shells out to the system `ufw` binary.
- Root privileges on the managed machine. The app uses `sudo` (see [Privileges](#privileges-and-sudo)).
- On Linux, a working OpenGL/EGL stack and X11 or Wayland session — present on any normal desktop. The app links these dynamically, so no extra packages are usually needed.

### Build prerequisites

- **Rust 1.85 or newer** (the crate uses the 2024 edition). Install via [rustup](https://rustup.rs):
  ```sh
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  ```
- **A C compiler** (`cc`/`gcc`/`clang`) and `pkg-config`. The TLS backend (ring) compiles a small amount of C.
- `git` if you clone the repository.

---

## Installation

### Option 1 — Build and run from source

```sh
git clone <your-repository-url> ufw-manager
cd ufw-manager
cargo build --release
./target/release/ufw-manager
```

### Option 2 — Install as a user command (`cargo install`)

Installs the binary into `~/.cargo/bin` (make sure that directory is on your `PATH`):

```sh
cargo install --path .
ufw-manager
```

### Option 3 — System-wide install with a desktop entry

Build a release binary, copy it to a system location, and register a launcher so it appears in your application menu.

```sh
# 1. Build
cargo build --release

# 2. Install the binary
sudo install -Dm755 target/release/ufw-manager /usr/local/bin/ufw-manager

# 3. Create a desktop entry
sudo install -Dm644 /dev/stdin /usr/share/applications/ufw-manager.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=UFW Manager
Comment=Manage UFW firewalls locally or over SSH
Exec=ufw-manager
Terminal=false
Categories=System;Security;
Keywords=firewall;ufw;security;
EOF
```

The app is a GUI, so launching it from the menu or a terminal both work. If you want it to be able to use `sudo` interactively the usual way, see below.

### Option 4 — Distro build dependencies

If `cargo build` fails, install the toolchain and graphics headers for your distribution.

**Debian / Ubuntu / Linux Mint**

```sh
sudo apt update
sudo apt install build-essential pkg-config git
# Only needed if the window or file dialogs fail to appear at runtime:
sudo apt install libgl1 libglx-mesa0 libegl1 libxkbcommon0 libwayland-client0 libx11-6
```

**Fedora**

```sh
sudo dnf install @development-tools pkgconf-pkg-config git
sudo dnf install mesa-libGL mesa-libEGL libxkbcommon
```

**Arch Linux / Manjaro**

```sh
sudo pacman -S --needed base-devel pkgconf git mesa libxkbcommon
```

**macOS** — install Xcode command line tools (`xcode-select --install`) and Rust via rustup. UFW is Linux-only, so on macOS the app is only useful for managing remote Linux servers.

**Windows** — install the [Visual Studio C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/) and Rust via rustup. As with macOS, only remote management applies.

---

## Quick start — local firewall

1. Make sure UFW is installed:

   ```sh
   sudo apt install ufw        # or your distro's package manager
   ```

2. Configure non-interactive privilege escalation (recommended), or plan to enter a sudo password in the app — see [Privileges](#privileges-and-sudo).

3. Launch the app:

   ```sh
   ufw-manager
   ```

4. The app starts on the **Local machine** endpoint and immediately refreshes the firewall state. Use the left sidebar to move between **Overview**, **Rules**, **Applications**, **Servers**, and **Command log**.

5. Common tasks:
   - **Enable/disable** the firewall and change **default policies** and **logging** from the Overview page.
   - **Add a rule** from the Rules page (`Add rule…`): pick action, direction, protocol, port(s), source/destination, interface, and an optional comment. The exact command is previewed before you commit; enable *Dry run* to see it without applying.
   - **Delete a rule** by selecting it in the table and clicking `Delete`.
   - **Allow/deny an application profile** from the Applications page.

Every applied change is recorded in the **Command log** with its output.

---

## Tutorial — managing a remote server

### 1. Prepare the remote server

On the server you want to manage, make sure UFW is installed and your user can escalate to root.

```sh
# Install UFW (Debian/Ubuntu example)
sudo apt update && sudo apt install ufw

# Create a dedicated key if you don't have one (run locally)
ssh-keygen -t ed25519
ssh-copy-id user@server.example.com
```

Choose one of the two privilege options below.

**A. Passwordless sudo (best for unattended use)** — add a drop-in on the *server*.

The app runs privileged operations as `sudo sh -c '<script>'` (a whole firewall refresh or import is a single script, so `sh` is the interpreter). Because of that, a sudoers rule scoped only to `/usr/sbin/ufw` is **not** enough — the rule must allow the interpreter the app calls. Pick the narrowest option that fits your threat model:

```sh
# On the server, as root. Replace "myuser" with the SSH login user.

# Most granular the app supports: allow that user to run sh via sudo.
echo 'myuser ALL=(root) NOPASSWD: /bin/sh' | sudo tee /etc/sudoers.d/ufw-manager

# Or, if you prefer a generalized rule:
# echo 'myuser ALL=(root) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ufw-manager

sudo chmod 0440 /etc/sudoers.d/ufw-manager
```

> Note: allowing `NOPASSWD: /bin/sh` effectively grants full root to that user (a shell can run anything). If that is unacceptable, use option B instead.

**B. Let the app hold the sudo password in memory** — leave sudo as-is. The app will prompt for the server's sudo password and pipe it to `sudo -S` (it is never written to disk). This needs no sudoers changes.

### 2. Add the server in the app

1. Open the **Servers** page and click `Add server…`.
2. Fill in:
   - **Label** — a friendly name (e.g. `production web`).
   - **Host** — IP address or hostname.
   - **Port** — SSH port (default `22`).
   - **User** — SSH login user.
   - **Authentication** — `SSH key` (optionally a specific key file) or `Password`.
   - **Privileges** — whether to run firewall commands with `sudo`.
   - **Host keys** — "Automatically accept new host keys" (accept-new policy) or verify against `~/.ssh/known_hosts`.
3. Click `Save`.

Server profiles are stored in `~/.config/ufw-manager/servers.json`. **Passwords and passphrases are never persisted** — they live only in memory for the current session.

### 3. Connect and manage

- Click `Connect` on the server row to establish the session, then select the server in the top endpoint dropdown to make it the active endpoint (or just pick it in the dropdown — the app will connect and refresh as part of that switch).
- The endpoint indicator in the top bar turns green when connected.
- All pages now operate on the remote firewall. Switch back to **Local machine** with the top dropdown at any time.
- If the app needs a password it can't get non-interactively, it shows a prompt, stores it in memory, and automatically retries the failed operation.

---

## Import & export

Export from the top bar (`Export…`) or from the Servers page (to include saved profiles). Import with `Import…`.

### JSON bundle

A structured, human-readable snapshot:

```json
{
  "format": "ufw-manager-export",
  "version": 1,
  "exported_at_unix": 1730000000,
  "firewall": {
    "active": true,
    "logging": "low",
    "defaults": { "incoming": "deny", "outgoing": "allow", "routed": "disabled" },
    "added_commands": [
      "ufw allow 22/tcp",
      "ufw deny from 10.0.0.0/8"
    ]
  },
  "servers": []
}
```

`servers` is optional and may contain saved server profiles (never passwords). Importing servers is opt-in.

### Shell script

A runnable `#!/bin/sh` script that recreates the firewall:

```sh
#!/bin/sh
# Exported by ufw-manager
# Run as root: sh this-file

ufw default deny incoming
ufw default allow outgoing
ufw default disabled routed
ufw logging low

# Rules (ufw show added)
ufw allow 22/tcp
ufw deny from 10.0.0.0/8

ufw --force enable
```

You can run it directly on a server (`sudo sh exported.sh`) or import it back into the app.

### Merge vs replace

- **Merge (default)** — applies defaults, logging level, and rules. It *never* enables or disables the firewall.
- **Replace (destructive)** — runs `ufw --force reset` first (deletes **all** current rules), then applies the file. The enabled/disabled state is restored from the file (`active` in JSON, or the script's `enable`/`disable` line); if the file does not specify one, the current state is kept.

Replace mode always asks for confirmation before running.

### Import safety

Only lines that begin with `ufw ` are accepted. Any other line — in a script or in a JSON `added_commands` list — is rejected with an error naming the offending line/command, so an imported file cannot smuggle arbitrary shell commands into a privileged run.

---

## Privileges and sudo

The app only escalates when it needs to, and prefers non-interactive modes first:

| Situation | Command used |
| --- | --- |
| Already running as root | the command as-is |
| No sudo password known | `sudo -n sh -c '<command>'` (fails fast if a password is required) |
| Sudo password provided in-app | `sudo -S -p '' sh -c '<command>'` with the password on stdin |
| Sudo not wanted for the server | the command as-is (expect UFW to refuse without root) |

When a command fails because a password is required or was rejected, the app detects it, shows a prompt, and retries automatically once you provide the password. Passwords and key passphrases are held in memory only.

---

## Security model

- **Credentials are never written to disk.** Sudo passwords, SSH passwords, and key passphrases are kept in an in-memory map for the session only.
- **Saved servers contain no secrets** — only host details and an optional path to a key file. They live in `~/.config/ufw-manager/servers.json`.
- **Host keys follow ssh's accept-new policy.** Unknown hosts can be accepted (optionally auto-learned into `known_hosts`); a *changed* host key is always rejected to protect against man-in-the-middle attacks.
- **Import validation** rejects anything that is not a `ufw` command (see above).
- **Destructive replace imports** are gated behind an explicit confirmation dialog.

---

## Troubleshooting

**"ERROR: You need to be root to run this script"**
UFW was invoked without privileges. Either run the app as root, configure passwordless sudo, or supply the sudo password when prompted.

**A sudo password prompt keeps coming back**
The password was rejected (or `sudo -n` failed). Check that the account is in `sudoers` and that the password is correct. For remote servers, make sure "Run firewall commands with sudo" is enabled for that profile.

**"No usable SSH key and no password supplied"**
No key from your configured/default paths worked and no password was available. Use the Servers dialog to set a key file, or switch the authentication method to Password and enter it when prompted.

**"Host key ... was rejected (unknown or changed)"**
The server's key is not in `known_hosts`, or it differs from the recorded one. Verify the server, then either add it to `~/.ssh/known_hosts` or enable "Automatically accept new host keys" for that server. If the key *changed*, investigate before accepting.

**Could not load an encrypted key**
Provide the key passphrase when prompted, or use `ssh-add`/an unencrypted key.

**Window does not appear / OpenGL errors on Linux**
Install the graphics stack (`libgl1`, `libegl1`, `libxkbcommon0`) and make sure `DISPLAY` or `WAYLAND_DISPLAY` is set. Under a minimal/WSL environment you may need a desktop session.

**File dialogs do not open**
The file picker relies on the platform's native dialog/portal. On a minimal Wayland session, ensure `xdg-desktop-portal` and a backend are installed.

---

## Development

```sh
cargo build          # debug build
cargo run            # run
cargo test           # unit tests (parsers, command builders, import/export)
cargo clippy --all-targets
cargo fmt
```

All UFW parsing and command construction is pure and unit-tested in `src/ufw.rs`; the import/export logic is tested in `src/io.rs`.

## Project layout

| File | Responsibility |
| --- | --- |
| `src/main.rs` | Entry point and window setup. |
| `src/app.rs` | Application state, update loop, dialogs, job dispatch. |
| `src/pages.rs` | The five pages (Overview, Rules, Applications, Servers, Log). |
| `src/ufw.rs` | UFW output parsing and command construction. |
| `src/worker.rs` | Background job runner (local sudo + remote exec). |
| `src/ssh.rs` | SSH connection, auth, known-hosts policy, command exec. |
| `src/io.rs` | Import/export formats and server-profile persistence. |
| `src/model.rs` | Shared types (servers, endpoints, policies). |
