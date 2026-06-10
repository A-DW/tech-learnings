# Antigravity IDE Manual

## Overview

This manual documents a user-local installation of Antigravity IDE from the official Linux `.tar.gz` archive, using the unpacked application directory and a custom `.desktop` launcher rather than a native package manager install. The approach is appropriate when Antigravity IDE is shipped as a raw tarball and the goal is to keep the install simple, reversible, and independent of root-owned package management.

In this installation model, the `antigravity-ide` entrypoint is not an ELF binary but a shell wrapper script, and on some Linux setups the IDE only launches reliably when the wrapper is invoked with `--no-sandbox`. This behavior is consistent with community reports around Linux tarball installs of Antigravity and other Chromium/Electron-based applications where the sandbox helper is not fully configured for that environment.

---

## Installation layout

The documented layout uses a renamed extracted folder called `Antigravity-IDE` stored under `~/Documents`, with the launcher script at `~/Documents/Antigravity-IDE/bin/antigravity-ide` and the icon at `~/Documents/Antigravity-IDE/resources/app/resources/linux/code.png`.

This is not a mandatory path; it is simply a user-chosen location that works as long as the `.desktop` file uses the correct absolute paths.

> ⚠️ `.desktop` files should always use full absolute paths in `Exec=` and `Icon=`. GNOME does **not** expand shell shortcuts like `~` inside launcher files.

---

## Why this installation method was chosen

Running Antigravity IDE directly from the extracted tarball is the lightest-weight deployment model: no root privileges are required, no system package database is modified, and uninstalling is as simple as removing the extracted directory and the user-local `.desktop` file. This is especially useful when upstream distributes portable archives instead of fully managed `.deb` or `.rpm` packages.

The trade-off is that the user becomes responsible for launcher integration, updates, and troubleshooting runtime flags such as `--no-sandbox`. In practice, this is often acceptable for advanced Linux users who prefer explicit control over where applications live and how they are invoked.

---

## Exact procedure followed

The sequence below reflects the practical installation and troubleshooting flow that was carried out.

### 1. Extract the official tarball

The official archive `Antigravity IDE.tar.gz` was extracted, producing a directory named `Antigravity IDE`.

```bash
tar xzf "Antigravity IDE.tar.gz"
```

**Result:**

```
A new directory named: Antigravity IDE
```

---

### 2. Rename the extracted directory

The extracted folder was renamed from `Antigravity IDE` to `Antigravity-IDE` to avoid spaces in paths when referencing the launcher and icon files.

```bash
mv "Antigravity IDE" "Antigravity-IDE"
```

**Result:** Directory renamed successfully; no output returned.

---

### 3. Move the directory into `~/Documents`

```bash
mv "Antigravity-IDE" ~/Documents/
```

**Result:** Directory moved successfully; no output returned.

---

### 4. Initial launch attempt

```bash
~/Documents/Antigravity-IDE/bin/antigravity-ide
```

**Result:** No visible output and no application window opened.

This showed the path was callable but the runtime was not successfully bringing up the IDE window. Troubleshooting was required.

---

### 5. Create a user-local desktop launcher

A `.desktop` file was created at `~/.local/share/applications/antigravity-ide.desktop` so Antigravity IDE would appear in the GNOME application menu.

```ini
[Desktop Entry]
Type=Application
Name=Antigravity IDE
Comment=Google Antigravity AI IDE
Exec=/home/YOURUSER/Documents/Antigravity-IDE/bin/antigravity-ide --no-sandbox %F
Icon=/home/YOURUSER/Documents/Antigravity-IDE/resources/app/resources/linux/code.png
Terminal=false
Categories=Development;IDE;TextEditor;
```

> Replace `YOURUSER` with your actual Linux username. Do **not** use `~` here.

---

### 6. Make the launcher executable and refresh the desktop database

```bash
chmod +x ~/.local/share/applications/antigravity-ide.desktop
update-desktop-database ~/.local/share/applications
```

**Result:** The Antigravity IDE launcher appeared in the GNOME application menu.

---

### 7. Attempt to launch from the GNOME application menu

**Result:** The launcher was visible in the menu, but clicking it did not open Antigravity IDE.

> Menu visibility only confirms the `.desktop` file is discoverable — it does not confirm the runtime behind `Exec=` is healthy.

---

### 8. Inspect GNOME user-session logs

To check whether GNOME was actually executing the launcher:

```bash
journalctl --user -f
```

**Observed output:**

```
gnome-character[12662]: JS LOG: Characters Application started
gpg-agent[3644]: can't connect to the daemon /usr/lib/gnupg/scdaemon: IPC connect call failed
systemd[2339]: Started app-gnome-antigravity\x2dide-12713.scope - Application launched by gnome-shell.
gpg-agent[3644]: can't connect to the daemon /usr/lib/gnupg/scdaemon: IPC connect call failed
gpg-agent[3644]: can't connect to the daemon /usr/lib/gnupg/scdaemon: IPC connect call failed
gnome-character[12662]: JS LOG: Characters Application exiting
```

**Interpretation:**

- The `systemd` scope line confirmed GNOME was attempting to launch Antigravity IDE via the `.desktop` file — ruling out a launcher indexing issue.
- The repeated `gpg-agent` / `scdaemon` lines are unrelated background noise (GnuPG smartcard support) and have no causal link to the Antigravity startup failure.

---

### 9. Verify the launcher file type and executable bit

```bash
ls -l ~/Documents/Antigravity-IDE/bin/antigravity-ide
file ~/Documents/Antigravity-IDE/bin/antigravity-ide
```

**Observed results:**

```
-rwxr-xr-x
a sh script, ASCII text executable
```

**Interpretation:**

- `-rwxr-xr-x` confirmed the file is executable.
- `a sh script, ASCII text executable` revealed that `antigravity-ide` is a **shell wrapper**, not a native ELF binary. The script bootstraps the actual embedded Electron/Chromium runtime and passes arguments to it. This changes how it should be debugged.

---

### 10. Retry the launcher with `--no-sandbox`

```bash
~/Documents/Antigravity-IDE/bin/antigravity-ide --no-sandbox
```

**Result:** ✅ Antigravity IDE opened successfully.

This was the decisive troubleshooting step. It proved the issue was not path resolution or `.desktop` indexing, but a **sandbox-related runtime failure** in the unpacked Linux install.

---

### 11. Update the `.desktop` launcher to match the working command

The `Exec=` line in `~/.local/share/applications/antigravity-ide.desktop` was updated to include `--no-sandbox`:

```ini
Exec=/home/YOURUSER/Documents/Antigravity-IDE/bin/antigravity-ide --no-sandbox %F
```

Then refreshed:

```bash
update-desktop-database ~/.local/share/applications
```

**Result:** ✅ The GNOME application menu entry now launches Antigravity IDE successfully.

---

## Why `--no-sandbox` was required

Linux tarball installs of Chromium/Electron applications depend on a sandbox helper (`chrome-sandbox`) that often requires root ownership and `4755` SUID permissions to work correctly. When installing via an unpacked user-local tarball (no native package manager), that configuration is not automatically applied.

When the wrapper script tries to initialize sandboxing and fails, the Chromium/Electron runtime aborts silently and exits — which is exactly the behaviour observed in step 4. Adding `--no-sandbox` bypasses the sandbox initialization entirely.

> **Security note:** `--no-sandbox` is a compatibility workaround, not an ideal production configuration. For a more hardened setup, the correct fix is to set proper ownership and permissions on the `chrome-sandbox` binary shipped inside the tarball:
>
> ```bash
> sudo chown root:root ~/Documents/Antigravity-IDE/chrome-sandbox
> sudo chmod 4755 ~/Documents/Antigravity-IDE/chrome-sandbox
> ```
>
> That restores proper sandboxing and removes the need for `--no-sandbox`.

---

## What `%F` does in the `.desktop` file

`%F` is a freedesktop Exec placeholder that expands to a list of file paths when the application is launched with files selected in a file manager. This allows one launcher definition to support "Open with Antigravity IDE" for multiple files at once.

When launched from the app menu with no file arguments, `%F` expands to nothing and has no effect on normal startup. It can be omitted if file-association behaviour is not needed.

---

## Common mistakes to avoid

| Mistake | Why it breaks things |
|--------|----------------------|
| Using `~` in `Exec=` or `Icon=` | GNOME does not expand shell shortcuts in `.desktop` files |
| Wrong directory name: `application` vs `applications` | Desktop environment ignores launchers outside the correct path |
| Not checking `file` on the entrypoint | Shell wrappers require different debugging approaches than ELF binaries |
| Assuming menu visibility = app health | They are independent; GNOME can index a launcher that points to a broken command |
| Omitting `--no-sandbox` in tarball installs | Chromium/Electron sandbox init fails silently in unpacked user-local installs |

---

## Final working state

```
~/Documents/Antigravity-IDE/          ← extracted and renamed tarball
    bin/antigravity-ide               ← shell wrapper (entry point)
    resources/app/resources/linux/
        code.png                      ← icon used in the launcher

~/.local/share/applications/
    antigravity-ide.desktop           ← user-local GNOME launcher
```

Launch command (manual):

```bash
~/Documents/Antigravity-IDE/bin/antigravity-ide --no-sandbox
```

Launch command (via `.desktop`):

```ini
Exec=/home/YOURUSER/Documents/Antigravity-IDE/bin/antigravity-ide --no-sandbox %F
```
