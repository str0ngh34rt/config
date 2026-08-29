# Filesystem Layout for Agents

This document defines the standard directory structure and setup procedure for agent-based development workflows.

## Directory Structure

All agent workspace data lives under the root projects directory (`/projects`). Each project gets its own directory containing a central bare Git repository (`bare.git/`) and a dedicated `worktrees/` directory. Temporary files and untracked artifacts live inside `.tmp/` within each specific worktree.

```text
/projects/
├── <project-name>/
│   ├── bare.git/             # Central bare Git repository
│   └── worktrees/
│       ├── main/             # Primary worktree (main branch)
│       │   └── .tmp/         # Temporary scratch space for main
│       ├── agent-task-1/     # Worktree for Task 1
│       │   └── .tmp/         # Temporary scratch space for Task 1
│       └── agent-task-2/     # Worktree for Task 2
│           └── .tmp/         # Temporary scratch space for Task 2
```

---

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
```

---

### 2. Project Setup & Bare Clone Initialization

Before creating worktrees, create the project container and initialize `bare.git/`.

#### Option A: Clone an Existing Repository

Clone a remote repository into `bare.git/`:

```bash
PROJECT_NAME="my-project"
REPO_URL="git@github.com:user/my-project.git"

mkdir -p "/projects/${PROJECT_NAME}/worktrees"
git clone --bare "${REPO_URL}" "/projects/${PROJECT_NAME}/bare.git"

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
git init --bare "/projects/${PROJECT_NAME}/bare.git"
```

---

### 3. Repository Exclude Configuration

Configure `bare.git/info/exclude` to ignore `.tmp/` across all present and future worktrees:

```bash
echo ".tmp/" >> "/projects/${PROJECT_NAME}/bare.git/info/exclude"
```

> **Note:** Ensure `.gitignore` inside the target repository also includes `.tmp/` to prevent accidental commits across environments.

---

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

---

## Worktree Verification

List active worktrees to verify the setup:

```bash
git --git-dir=/projects/${PROJECT_NAME}/bare.git worktree list
```
