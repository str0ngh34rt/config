# Filesystem Layout for Agents

Running local AI agents inside your personal home directory exposes sensitive assets to unprivileged execution. Unsanitized prompt outputs or malicious tools can read your SSH keys, personal configurations, and browser credentials. 

This document defines a secure filesystem layout and permission model for agent workloads. The layout isolates repositories under `/projects` and uses POSIX Access Control Lists (ACLs) to manage access between your primary account and an unprivileged agent user.

## Overview

The layout divides project responsibilities between two user roles operating within a shared `projects` group.

| Feature / Responsibility | Primary Human User (`$USER`)                        | Agent Runner (`agent`)                                 |
| ------------------------ | --------------------------------------------------- | ------------------------------------------------------ |
| **Account Type**         | Primary interactive user account                    | Unprivileged system user account                       |
| **Directory Scope**      | Full access to home directory and `/projects`       | Access restricted exclusively to `/projects`           |
| **Git Operations**       | Clones `bare.git/`, manages branches, pushes remote | Works within assigned `worktrees/` branches            |
| **Task Execution**       | Direct terminal and interactive tool usage          | Automated workflows and Rootless Podman containers     |
| **Scratchpad Space**     | Shared workspace access                             | Writes logs, plans, and state to local `.tmp/`         |

## Directory Structure

All agent workspace data lives under the root projects directory (`/projects`). Each project gets its own directory containing a central bare Git repository (`bare.git/`) and a dedicated `worktrees/` directory. Temporary files and untracked artifacts live inside `.tmp/` within each specific worktree.

```text
/projects/
└── <project-name>/
    ├── bare.git/             # Central bare Git repository
    └── worktrees/
        ├── main/             # Primary worktree (main branch)
        │   └── .tmp/         # Temporary scratch space for main
        ├── agent-task-1/     # Worktree for Task 1
        │   └── .tmp/         # Temporary scratch space for Task 1
        └── agent-task-2/     # Worktree for Task 2
            └── .tmp/         # Temporary scratch space for Task 2
```

## Setup Instructions

### 1. Base Setup & Permissions

Create the root directory for all projects and assign shared access to your user and the agent user.

```bash
# Define user and group variables
HUMAN_USER="$USER"
AGENT_USER="agent"
SHARED_GROUP="projects"

# Create shared group and add users
sudo groupadd -f "${SHARED_GROUP}"
sudo usermod -aG "${SHARED_GROUP}" "${HUMAN_USER}"
sudo usermod -aG "${SHARED_GROUP}" "${AGENT_USER}"

# Create root directory and set group ownership
sudo mkdir -p /projects
sudo chown -R :"${SHARED_GROUP}" /projects

# Set permissions and setgid bit (new files inherit group ownership)
sudo chmod 2775 /projects

# Set access and default ACLs (new files inherit group write permission)
sudo setfacl -R -m g:"${SHARED_GROUP}":rwX /projects
sudo setfacl -R -d -m g:"${SHARED_GROUP}":rwx /projects
```

> **Note:** The setgid bit and the default ACL do different jobs. The setgid bit gives each new file the
> `projects` group. It does not give the group write permission. The default ACL gives that permission. Use both.

The default ACL sets the maximum permissions for a new file. The mode of the process that creates the file
reduces them. With the default umask `022`, Git creates files with mode `0644`. This masks the group entry down
to read only, and the ACL has no effect. Set umask `002` for both accounts:

```bash
for U in "${HUMAN_USER}" "${AGENT_USER}"; do
  echo "umask 002" | sudo tee -a /home/"${U}"/.bashrc
done
```

> **Note:** `run-as.md` starts the agent with `sudo -i`, which reads `.bashrc` through the login profile. Confirm
> the value in the agent session with `umask`. The expected output is `0002`.

Verify the result. Create a file as one user and write to it as the other:

```bash
sudo -u "${AGENT_USER}" touch /projects/acl-test
sudo -u "${HUMAN_USER}" tee -a /projects/acl-test <<< "write test"
sudo rm /projects/acl-test
```

A `Permission denied` error means the umask is still `022`. Open a new login shell and try again.

### 2. Project Setup & Bare Clone Initialization

Before creating worktrees, create the project container and initialize `bare.git/`.

#### Option A: Clone an Existing Repository

Clone a remote repository into `bare.git/`:

```bash
PROJECT_NAME="my-project"
REPO_URL="git@github.com:user/my-project.git"

mkdir -p "/projects/${PROJECT_NAME}/worktrees"
git clone --bare --config core.sharedRepository=group "${REPO_URL}" "/projects/${PROJECT_NAME}/bare.git"

# Configure fetch refspec for worktree branch tracking
cd "/projects/${PROJECT_NAME}/bare.git"
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git fetch origin
```

#### Option B: Initialize a New Local Project

Initialize a new empty bare repository:

```bash
PROJECT_NAME="my-project"

mkdir -p "/projects/${PROJECT_NAME}/worktrees"
git init --bare --shared=group "/projects/${PROJECT_NAME}/bare.git"
```

> **Note:** `core.sharedRepository=group` makes Git write group-writable objects and refs inside `bare.git/`.
> Without it, a repack by one user creates files that the other user cannot replace.

### 3. Repository Exclude Configuration

Configure `bare.git/info/exclude` to ignore `.tmp/` across all present and future worktrees:

```bash
echo ".tmp/" >> "/projects/${PROJECT_NAME}/bare.git/info/exclude"
```

> **Note:** Ensure `.gitignore` inside the target repository also includes `.tmp/` to prevent accidental commits across environments.

### 4. Worktree Setup

Create individual worktrees inside `worktrees/` pointing to `bare.git/`.

#### Create Main Branch Worktree

```bash
cd "/projects/${PROJECT_NAME}"
git --git-dir=bare.git worktree add worktrees/main main
mkdir -p worktrees/main/.tmp
```

#### Create Task-Specific Worktrees

```bash
cd "/projects/${PROJECT_NAME}"
git --git-dir=bare.git worktree add worktrees/agent-task-1 -b feature/task-1
mkdir -p worktrees/agent-task-1/.tmp
```
