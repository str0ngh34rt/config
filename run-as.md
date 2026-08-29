# How to Run a Desktop Application as Another User on Ubuntu

This guide details how to configure Ubuntu to run any desktop application under a separate, isolated system user account without requiring password prompts when launched from the desktop application menu.

---

## Overview

Running applications under a separate user account provides security isolation and allows running distinct application instances (such as separate login profiles or browser contexts). This setup leverages:

1. **`xhost`** to grant display access to the X11 session.
2. **`sudo` with custom rules** for passwordless execution from the GUI desktop launcher.
3. **`XDG_RUNTIME_DIR` and D-Bus session configuration** in the target user's shell profile to support desktop integration and IPC.
4. **Desktop launchers (`.desktop`)** with background log file redirection.

---

## Prerequisites

Replace the following placeholders throughout the instructions:

* `<MAIN_USER>`: Your primary Ubuntu username (e.g., `strongheart`).
* `<TARGET_USER>`: The separate user account running the application (e.g., `codex`).
* `<APP_NAME>`: The command/binary name of the app (e.g., `chatgpt`, `claude`, `google-chrome`).
* `<SCRIPT_NAME>`: Desired name for your launcher script (e.g., `run-as-codex`).

---

## Step 1: Prepare Target User Environment

1. Ensure the target user account exists and has access to graphics drivers:
   ```bash
   sudo usermod -aG render,video <TARGET_USER>
   ```

2. Enable lingering so D-Bus user services run automatically in the background for the target user:
   ```bash
   sudo loginctl enable-linger <TARGET_USER>
   ```

3. Configure runtime directory exports in `<TARGET_USER>`'s shell environment. Append the following lines to `/home/<TARGET_USER>/.bashrc`:
   ```bash
   # User runtime & D-Bus session configuration
   export XDG_RUNTIME_DIR="/run/user/$(id -u)"
   export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
   ```

---

## Step 2: Create the Wrapper Script

Create a launcher script at `/usr/local/bin/<SCRIPT_NAME>`:

```bash
sudo nano /usr/local/bin/<SCRIPT_NAME>
```

Paste the following script content:

```bash
#!/bin/bash

# Configuration
TARGET_USER="<TARGET_USER>"
APP_EXEC="<APP_NAME>"

# 1. Allow target user to access current X display
xhost +si:localuser:"$TARGET_USER" > /dev/null

# 2. Launch application in target user's session with log redirection
sudo -i -u "$TARGET_USER" --preserve-env=DISPLAY,XAUTHORITY bash -c "$APP_EXEC >> \"\$HOME/${APP_EXEC}-app.log\" 2>&1"

# 3. Revoke access on application exit
xhost -si:localuser:"$TARGET_USER" > /dev/null
```

Make the launcher script executable:
```bash
sudo chmod +x /usr/local/bin/<SCRIPT_NAME>
```

---

## Step 3: Configure Passwordless Execution

Grant your primary user permission to run the wrapper script via `sudo` without entering a password.

1. Create a dedicated sudoers rule file:
   ```bash
   sudo visudo -f /etc/sudoers.d/<SCRIPT_NAME>
   ```

2. Add the following rule:
   ```text
   <MAIN_USER> ALL=(ALL) NOPASSWD: /usr/local/bin/<SCRIPT_NAME>
   ```

---

## Step 4: Create Desktop Launcher

Create a `.desktop` shortcut in your user's application menu directory:

```bash
nano ~/.local/share/applications/<TARGET_USER>-<APP_NAME>.desktop
```

Paste the following desktop entry configuration:

```ini
[Desktop Entry]
Version=1.0
Type=Application
Name=<APP_NAME> (<TARGET_USER>)
Comment=Launch <APP_NAME> isolated under <TARGET_USER> user
Exec=sudo /usr/local/bin/<SCRIPT_NAME>
Icon=<APP_NAME>
Terminal=false
Categories=Utility;Development;
```

---

## Monitoring Application Logs

Because the GUI process runs detached from standard terminal output, stdout and stderr are automatically appended to a log file inside the target user's home directory.

To inspect logs in real time:
```bash
tail -f /home/<TARGET_USER>/<APP_NAME>-app.log
```
