# Filesystem Layout for Agents

This document defines the standard directory structure and setup procedure for agent-based development workflows.

## Directory Structure

All agent workspace data lives under the root projects directory (`/projects`). Each project gets its own directory containing a central bare Git repository (`.git`), individual worktrees, and a temporary scratch space (`.tmp/`) for uncommitted state.

```text
/projects/
├── <project-name>/
│   ├── .git/                 # Central bare Git repository
│   ├── .tmp/                 # Project-wide temporary scratch space
│   ├── main/                 # Primary worktree (main branch)
│   ├── agent-task-1/         # Dedicated worktree for Task 1
│   └── agent-task-2/         # Dedicated worktree for Task 2
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

### 2. Project Setup and Bare Clone Initialization

Before creating worktrees, create the project directory and initialize the bare Git repository in `.git`.

#### Option A: Clone an Existing Repository

Clone a remote repository into the `.git` directory:

```bash
PROJECT_NAME="my-project"
REPO_URL="git@github.com:user/my-project.git"

mkdir -p "/projects/${PROJECT_NAME}"
git clone --bare "${REPO_URL}" "/projects/${PROJECT_NAME}/.git"

# Configure fetch refspec for worktree branch tracking
cd "/projects/${PROJECT_NAME}/.git"
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git fetch origin
```

#### Option B: Initialize a New Local Project

Initialize a new empty bare repository:

```bash
PROJECT_NAME="my-project"

mkdir -p "/projects/${PROJECT_NAME}/.git"
git init --bare "/projects/${PROJECT_NAME}/.git"
```

---

### 3. Temporary Scratch Space (`.tmp/`) & Exclude Rules

Create a `.tmp/` directory at the project root for temporary files, build outputs, or untracked agent artifacts. Ensure local Git settings ignore this path across all worktrees.

```bash
# Create the scratch space directory
mkdir -p "/projects/${PROJECT_NAME}/.tmp"

# Ignore .tmp/ globally across all project worktrees via repository exclude
echo ".tmp/" >> "/projects/${PROJECT_NAME}/.git/info/exclude"
```

> **Note:** To prevent local worktree instances from committing `.tmp/` if referenced inside individual working trees, ensure `.gitignore` in the target repository also includes `.tmp/`.

---

### 4. Worktree Setup

After setting up `.git` and configuration rules, create individual worktrees for the primary branch and agent tasks.

#### Create Main Branch Worktree

```bash
cd "/projects/${PROJECT_NAME}"
git --git-dir=.git worktree add main main
```

#### Create Task-Specific Worktrees

```bash
cd "/projects/${PROJECT_NAME}"
git --git-dir=.git worktree add agent-task-1 -b feature/task-1
```

---

## Worktree Verification

List active worktrees to verify the setup:

```bash
git --git-dir=.git worktree list
```
