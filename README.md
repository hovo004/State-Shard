# state-shard

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Bash](https://img.shields.io/badge/bash-4%2B-4EAA25?logo=gnu-bash&logoColor=white)](#requirements)
[![Terraform](https://img.shields.io/badge/terraform-required-844FBA?logo=terraform&logoColor=white)](#requirements)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](./CONTRIBUTING.md)

> Created by **Hovhannes Hovhannisyan**

A bash script to safely migrate Terraform resources from one remote state to another — without using `import` blocks or manual `terraform state mv` commands.

> ⚠️ **This script mutates remote Terraform state.** Always test on a non-production state first, and keep the backups it creates until you've confirmed the migration with `terraform plan`.

---

## Table of Contents

- [The Problem](#the-problem)
  - [Real-world example](#real-world-example)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
  - [Flags](#flags)
- [Input File Format](#input-file-format)
- [What the Script Does — Step by Step](#what-the-script-does--step-by-step)
- [Example Run](#example-run)
- [Backups](#backups)
- [After the Migration](#after-the-migration)
- [Limitations](#limitations)
- [Contributing](#contributing)
- [License](#license)

---

## The Problem

As infrastructure grows, teams often end up with a single, monolithic Terraform state managing everything — sometimes thousands of resources in one module. This creates real pain:

- **Slow `plan`/`apply`** — Terraform has to load and compare the *entire* state, even when only a handful of resources actually changed.
- **Long CI/CD pipelines** — every pipeline run pays the cost of the full state, regardless of what was touched.
- **Poor isolation** — services and environments are tangled together in one state, making ownership and blast-radius control harder than it should be.

The fix is well known: split the state by service and/or environment, so each module only manages the resources it actually owns. The tricky part is *migrating* the existing resources into the new states without breaking anything.

That migration is normally painful:
- Writing `import` blocks requires knowing every resource's import ID.
- Running `terraform state mv` manually for dozens (or hundreds) of resources is slow and error-prone.
- One mistake in the process can corrupt your state.

**state-shard** solves this: you provide a list of resource addresses, and it handles pulling, moving, verifying, and pushing the states for you — safely and interactively, pausing for your confirmation at every major step.

### Real-world example

One real migration looked like this:

**Before** — a single module managing everything:
```text
all-vms-conf/
└── main.tf
```
This module contained **1,700+ Terraform resources**. Every `plan` and `apply` became increasingly slow, since Terraform had to process the full state even for a tiny change.

**After** — split by service and environment:
```text
mongo-vms/
├── dev/main.tf
├── stage/main.tf
└── prod/main.tf
postgres-vms/
├── dev/main.tf
├── stage/main.tf
└── prod/main.tf
redis-vms/
├── dev/main.tf
├── stage/main.tf
└── prod/main.tf
```
Each module now owns its own state. state-shard was used to move the relevant resources out of the original 1,700+ resource state into each of the new, smaller states.

**Result:** CI/CD pipeline execution time dropped by roughly **3–4×**, since each pipeline now only processes the state for the affected service and environment — plus smaller state files, better isolation, and easier ownership overall.

---

## How It Works

All operations happen locally. The remote backends are only touched at the very end when you explicitly confirm.

Works with any Terraform-supported backend — GitLab, S3, GCS, Azure Blob, Terraform Cloud, and others.

```
Remote backend (source)          Remote backend (target)
        │                                  │
        │  state pull                      │  state pull
        ▼                                  ▼
./source_TIMESTAMP.tfstate       ./target_TIMESTAMP.tfstate
        │                                  │
        │         state mv (local)         │
        └──────── moves resources ────────▶│
        │                                  │
        │  state push                      │  state push
        ▼                                  ▼
source backend (resources gone)  target backend (resources added)
```

---

## Prerequisites

Before running the script, make sure the following are in place.

### 1. Tooling

- `bash` 4+ — the script uses `mapfile`, which requires bash 4+
  - **macOS:** default bash is 3.2 — install a newer version via Homebrew: `brew install bash`
  - **Linux:** bash 4+ is typically already available at `/bin/bash` or `/usr/bin/bash`
  - The script uses `#!/usr/bin/env bash` which automatically finds the correct bash in your `PATH` on any system
- `terraform`
- `jq`

### 2. Set up the target Terraform root

Add a backend block to your target module pointing to a new state. The tool works with any Terraform-supported backend. Examples:

**GitLab:**
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

**S3:**
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "new-root/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Then initialise it:
```bash
cd ./target
terraform init
```

### 3. Write your resource configurations

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

### Uninstall

```bash
sudo rm /usr/local/bin/state-shard
```

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

> 💡 **Tip:** For `for_each`/`count` resources, run `terraform state list` on the source and copy the addresses exactly as shown (including the `["key"]` or `[index]` suffix) — no special handling needed.

---

## What the Script Does — Step by Step

| Step | Action |
|------|--------|
| **Pre-flight** | Shows the migration plan and asks for confirmation |
| **Step 1** | Confirms you have set up the target config and run `terraform init` |
| **Step 2** | Pulls both source and target states to local files and creates `.bak` backups |
| **Step 3** | Moves resources between the local state files using `terraform state mv` |
| **Step 4** | Saves the source and target state lists to `.txt` files for review — pauses for your confirmation |
| **Step 5** | Pushes both modified states back to their remote backends |

The script pauses at every major step and waits for you to press `[ENTER]` before continuing. You can abort at any point before the push.

---

## Example Run

A typical interactive session looks like this:

```text
$ state-shard -s ./source -t ./target -r resources.txt

Migration plan:
  Source: ./source
  Target: ./target
  Resources to move: 3

Press [ENTER] to continue, or Ctrl+C to abort...

[Step 1] Confirm target is initialised (terraform init has been run)? [y/N]: y
[Step 2] Pulling source state...        done
          Pulling target state...        done
          Backups saved:
            ./source_20260712153000.tfstate.bak
            ./target_20260712153000.tfstate.bak
[Step 3] Moving resources...
          module.vpc.aws_vpc.main -> module.networking.aws_vpc.main   OK
          (2 more...)                                                 OK
[Step 4] State lists saved for review:
            ./source_list_20260712153000.txt
            ./target_list_20260712153000.txt
          Review the files above. Press [ENTER] to push, or Ctrl+C to abort...

[Step 5] Pushing source state...        done
          Pushing target state...        done

Migration complete. Run `terraform plan` in both roots to verify.
```

---

## Backups

Before any modification, the script saves local backups of both states in the directory where you run the script:

```
./source_TIMESTAMP.tfstate.bak   ← original source state
./target_TIMESTAMP.tfstate.bak   ← original target state
```

The state list snapshots (from Step 4) are also saved in the same directory:
```
./source_list_TIMESTAMP.txt   ← source resources after move
./target_list_TIMESTAMP.txt   ← target resources after move
```

If something goes wrong after the push, you can restore either state manually:
```bash
terraform -chdir=./source state push ./source_TIMESTAMP.tfstate.bak
terraform -chdir=./target state push ./target_TIMESTAMP.tfstate.bak
```

---

## After the Migration

Run `terraform plan` in both roots to verify there are no unexpected changes:

```bash
terraform -chdir=./source plan
terraform -chdir=./target plan
```

Both should show **No changes** if the resource configurations in both modules match the migrated state entries.

---

## Limitations

- The script moves state entries only — it does not modify your `.tf` configuration files. You are responsible for having matching resource/module blocks in the target root before running it.
- Cross-workspace migrations (Terraform Cloud/Enterprise workspaces) have not been extensively tested — verify carefully with `terraform plan` after migrating.
- The script assumes both source and target backends are reachable and that you have valid credentials/auth configured locally for both.
- Large state files may take noticeably longer during the `state pull`/`state push` steps, depending on your backend.

---

## Contributing

Bug reports, feature requests, and pull requests are welcome — see [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines before submitting.

---

## License

Licensed under the [MIT License](./LICENSE).