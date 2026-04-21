# state-shard

> Created by **Hovhannes Hovhannisyan**

A bash script to safely migrate Terraform resources from one remote state to another — without using `import` blocks or manual `terraform state mv` commands.

---

## Why This Script?

When you have a large Terraform state file and want to split it, the standard approach is painful:
- Writing `import` blocks requires knowing every resource's import ID
- Running `terraform state mv` manually for dozens of resources is error-prone
- One mistake can corrupt your state

This script lets you simply provide a list of resource addresses, and it handles everything — pulling, moving, verifying, and pushing — safely and interactively.

---

## How It Works

All operations happen locally. The remote backends are only touched at the very end when you explicitly confirm.

```
GitLab (source backend)          GitLab (target backend)
        │                                  │
        │  state pull                      │  state pull
        ▼                                  ▼
/tmp/source_TIMESTAMP.tfstate    /tmp/target_TIMESTAMP.tfstate
        │                                  │
        │         state mv (local)         │
        └──────── moves resources ────────▶│
        │                                  │
        │  state push                      │  state push
        ▼                                  ▼
GitLab (source — resources gone) GitLab (target — resources added)
```

---

## Before You Run

You must do these two things manually before running the script:

**1. Set up the target Terraform root**

Add a backend block to your target module's `main.tf` (or equivalent) pointing to a new GitLab state address:

```hcl
terraform {
  backend "http" {
    address        = "https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/terraform/state/your-new-state-name"
    lock_address   = "https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/terraform/state/your-new-state-name/lock"
    unlock_address = "https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/terraform/state/your-new-state-name/lock"
    lock_method    = "POST"
    unlock_method  = "DELETE"
    retry_wait_min = "5"
  }
}
```

Then initialise it:
```bash
cd ./target
terraform init
```

**2. Write your resource configurations**

The script does not run `terraform plan` or `terraform apply`. It only moves state entries. Make sure your target module already has the resource/module configurations declared — after migration, `terraform plan` in both roots should show **No changes**, because the state and the configuration will be in sync.

---

## Installation

**Using curl:**
```bash
curl -fsSL https://raw.githubusercontent.com/hovo004/State-Shard/main/state-shard -o state-shard
chmod +x state-shard
sudo mv state-shard /usr/local/bin/state-shard
```

**Using wget:**
```bash
wget -q https://raw.githubusercontent.com/hovo004/State-Shard/main/state-shard -O state-shard
chmod +x state-shard
sudo mv state-shard /usr/local/bin/state-shard
```

After installation, run from anywhere:
```bash
state-shard -h
```

---

## Requirements

- `bash` 5+ — the script uses `mapfile` which requires bash 4+
  - **macOS:** default bash is 3.2 — install a newer version via Homebrew: `brew install bash`
  - **Linux:** bash 5+ is typically already available at `/bin/bash` or `/usr/bin/bash`
  - The script uses `#!/usr/bin/env bash` which automatically finds the correct bash in your `PATH` on any system
- `terraform`
- `jq`

---

## Usage

```bash
chmod +x state-shard

# Same-name move (resource keeps its address in target)
./state-shard \
  -s /path/to/source/terraform \
  -t /path/to/target/terraform \
  -r resources.txt

# Move with renaming (resource gets a new address in target)
./state-shard \
  -s /path/to/source/terraform \
  -t /path/to/target/terraform \
  -r resources.txt \
  -n new_names.txt
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `-s` | ✅ | Path to the **source** Terraform root (existing state) |
| `-t` | ✅ | Path to the **target** Terraform root (new backend) |
| `-r` | ✅ | Resources file — one old address per line |
| `-n` | ❌ | New names file — one new address per line, same line count as `-r` |
| `-h` | ❌ | Show help |

---

## Input File Format

### `-r` Resources file

One Terraform resource address per line. These are the addresses exactly as they appear in `terraform state list` on the source:

```
module.vpc.aws_vpc.main
module.eks.aws_eks_cluster.this
module.apply_topic.kubernetes_manifest.apply_topic["test-topic-1"]
```

### `-n` New names file (optional)

One new address per line. Line N in this file is the new name for line N in the resources file. Use this when you need to rename a resource as part of the migration:

```
module.networking.aws_vpc.main
module.kubernetes.aws_eks_cluster.this
module.apply_topic.kubernetes_manifest.apply_topic["test-topic-1-new"]
```

If `-n` is omitted, every resource keeps its original address in the target state.

---

## What the Script Does — Step by Step

| Step | Action |
|------|--------|
| **Pre-flight** | Shows the migration plan and asks for confirmation |
| **Step 1** | Confirms you have set up the target config and run `terraform init` |
| **Step 2** | Pulls both source and target states to local files and creates `.bak` backups |
| **Step 3** | Moves resources between the local state files using `terraform state mv` |
| **Step 4** | Shows you the target state list for visual review — pauses for your confirmation |
| **Step 5** | Pushes both modified states back to their remote backends |

The script pauses at every major step and waits for you to press `[ENTER]` before continuing. You can abort at any point before the push.

---

## Backups

Before any modification, the script saves local backups of both states:

```
/tmp/source_TIMESTAMP.tfstate.bak   ← original source state
/tmp/target_TIMESTAMP.tfstate.bak   ← original target state
```

If something goes wrong after the push, you can restore either state manually:
```bash
terraform -chdir=./source state push /tmp/source_TIMESTAMP.tfstate.bak
terraform -chdir=./target state push /tmp/target_TIMESTAMP.tfstate.bak
```

---

## After the Migration

Run `terraform plan` in both roots to verify there are no unexpected changes:

```bash
terraform -chdir=./source plan
terraform -chdir=./target plan
```

Both should show **No changes** if the resource configurations in both modules match the migrated state entries.
