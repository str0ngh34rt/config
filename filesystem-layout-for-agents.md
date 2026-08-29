# How to Set Up a Shared Multi-Agent Workspace Layout on Ubuntu

This guide details how to establish a clean, collaborative directory structure and permission model for hosting git repositories and worktrees shared between a primary human account and unprivileged AI agent runner accounts.

---

## Overview

When working alongside local AI agents, the goal is to grant agents full access to project repositories while protecting your personal home directory, SSH keys, and browser credentials. This configuration uses:

1. **A dedicated shared group (`developers`)** with POSIX Access Control Lists (ACLs) for automatic file permission inheritance.
2. **A standardized root layout (`/projects/`)** outside personal home directories.
3. **An isolated, unprivileged system account (`agent`)** designed to execute agent tasks and run containerized workloads (e.g., via Rootless Podman).
4. **Worktree-isolated temporary directories (`.tmp/`)** for session scratchpads, task plans, and local logs.

---

## Prerequisites

Replace the following placeholders if your environment uses different account names:

* `<PRIMARY_USER>`: Your primary Ubuntu login username (example: `strongheart`).
* `<AGENT_USER>`: The unprivileged system user running agent tasks (example: `agent`).
* `<SHARED_GROUP>`: The collaboration group (example: `developers`).

---

## Step 1: Create the Shared Group & Users

1. Create the `developers` group:
   ```bash
   sudo groupadd developers
   ```

2. Add your primary user and the agent user to the group:
   ```bash
   sudo usermod -aG developers strongheart
   sudo usermod -aG developers agent
   ```

3. Ensure the agent account has access to system rendering and virtualization groups:
   ```bash
   sudo usermod -aG render,video developers agent
   ```

4. Refresh your current terminal group membership:
   ```bash
   newgrp developers
   ```

---

## Step 2: Establish the Workspace Directory Structure

Keep all shared repositories and active worktrees contained within `/projects/`. 

```text
/projects/
├── <project-name>/
│   ├── bare.git/             # Bare repository database
│   └── worktrees/            # Active branch worktrees
│       ├── main/             # Main branch worktree (human/primary)
│       │   └── .tmp/         # Worktree-scoped scratchpad & build logs
│       ├── feature-a/        # Feature worktree (agent A)
│       │   └── .tmp/         # Worktree-scoped scratchpad & build logs
│       └── feature-b/        # Feature worktree (agent B)
│           └── .tmp/         # Worktree-scoped scratchpad & build logs
```

Create the root projects directory:
```bash
sudo mkdir -p /projects
```

---

## Step 3: Configure POSIX ACLs for Permission Inheritance

To prevent agents or IDEs from creating files that lock out the other user, configure POSIX ACL default inheritance rules on `/projects/`. Any file or directory created within this tree will automatically inherit read, write, and execute permissions for the `developers` group.

1. Set directory ownership and the setgid bit (`2775`):
   ```bash
   sudo chown -R strongheart:developers /projects
   sudo chmod -R 2775 /projects
   ```

2. Apply default POSIX ACLs to enforce group write inheritance on all future child items:
   ```bash
   sudo setfacl -d -m g:developers:rwx /projects
   sudo setfacl -d -m u:strongheart:rwx /projects
   ```

---

## Step 4: Configure Git & Defense-in-Depth Excludes

When initializing or cloning repositories under `/projects/`, configure Git to maintain group-write privileges on objects and ref updates. 

### 1. Enable Shared Repository Mode
```bash
cd /projects/<project-name>/bare.git
git config core.sharedRepository group
```

Ensure that both your local environment and the `agent` account operate with a `umask` of `002` when working inside this directory so new files remain group-writable by default (`rw-rw-r--` / `rwxrwxr-x`).

### 2. Standardize Repository `.gitignore`
Add `.tmp/` to your project's committed `.gitignore` file so all worktrees automatically exclude temporary agent files:
```gitignore
# Temporary agent scratchpads, plans, and build outputs
.tmp/
```

### 3. Apply Local Defense-in-Depth Exclusion
To ensure an agent cannot commit `.tmp/` files even if it modifies or deletes `.gitignore` inside its worktree, add `.tmp/` to the unversioned repository database exclude file:

```bash
echo ".tmp/" >> /projects/<project-name>/bare.git/info/exclude
```

---

## Step 5: (Optional) Isolate Agents via Rootless Podman

To restrict an agent session so it cannot view files outside its designated project worktree, run the agent process inside a Rootless Podman container scoped to that directory.

Example launch command for an agent session bound to `feature-a`:

```bash
podman run --rm -it \
  --user 1000:1000 \
  --userns keep-id \
  --volume /projects/hades/worktrees/feature-a:/workspace:Z \
  --workdir /workspace \
  agent-dev-image:latest
```

* `--userns keep-id`: Aligns container user IDs with host user IDs so created files remain seamlessly accessible to your IDE running under `strongheart`.
* `--volume`: Binds only the target worktree directory into the container environment, preventing access to host system files or `/home/strongheart`.

---

## Verification

Verify that permissions and group inheritance are functioning correctly:

1. Check directory ACL settings:
   ```bash
   getfacl /projects
   ```
2. Test file creation from the `agent` account:
   ```bash
   sudo -u agent touch /projects/test-file
   ls -l /projects/test-file
   ```
   *The test file should reflect ownership by `agent:developers` with group write permissions (`-rw-rw-r--+`), allowing `strongheart` to edit or delete it without `sudo`.*
