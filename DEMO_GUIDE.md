# Terraform Refresh vs Plan vs Refresh-Only: Complete Practical Guide

## Table of Contents

1. [The Core Concept](#1-the-core-concept)
2. [Command Comparison Table](#2-command-comparison-table)
3. [Setup: Create Drift (Out-of-Band Change)](#3-setup-create-drift)
4. [Demo 1: `terraform plan`](#4-demo-1-terraform-plan)
5. [Demo 2: `terraform refresh`](#5-demo-2-terraform-refresh)
6. [Demo 3: `terraform plan -refresh-only`](#6-demo-3-terraform-plan--refresh-only)
7. [Demo 4: `terraform apply -refresh-only`](#7-demo-4-terraform-apply--refresh-only)
8. [Summary: When to Use What](#8-summary-when-to-use-what)

---

## 1. The Core Concept

Terraform always works with **three sources of truth**:

```
+---------------------+      +---------------------+      +---------------------+
|   CONFIGURATION     |      |       STATE          |      |     REAL WORLD      |
|   (your .tf files)  |      |  (terraform.tfstate) |      |  (actual AWS infra) |
+---------------------+      +---------------------+      +---------------------+
         |                            |                            |
  "What you WANT"           "What Terraform             "What ACTUALLY
                             THINKS exists"              EXISTS in AWS"
```

### What is "Drift"?

Drift happens when someone changes infrastructure **outside** of Terraform — via the
AWS Console, AWS CLI, another automation tool, etc. This causes the **Real World** and
the **State File** to disagree.

### What is "Refresh"?

Refresh = Terraform **queries the real-world provider** (AWS) to read the current state
of every resource, and compares it with what's stored in the state file.

---

## 2. Command Comparison Table

| Command | Reads Real World? | Updates State File? | Plans Infra Changes? | Deprecated? |
|---|---|---|---|---|
| `terraform refresh` | Yes | **YES (immediately, no confirmation)** | No | **YES** |
| `terraform plan` | Yes (refresh first) | **No** (in-memory only) | **YES** (full plan) | No |
| `terraform plan -refresh-only` | Yes | **No** (in-memory only) | **No** (only shows drift) | No |
| `terraform apply -refresh-only` | Yes | **YES (after confirmation)** | **No** (only updates state) | No |

---

## 3. Setup: Create Drift

Your current infrastructure:
- **S3 bucket**: `terraform-refresh-demo-0f9a73bf`
- **Config tags**: Name, Environment, ManagedBy (3 tags)
- **State tags**: Name, Environment, ManagedBy (3 tags — matches config)
- **AWS tags**: Name, Environment, ManagedBy (3 tags — matches state)

### Step 1: Verify everything is in sync

```bash
terraform plan
```

You should see: **"No changes. Your infrastructure matches the configuration."**

### Step 2: Create drift — add a tag outside Terraform via AWS CLI

```bash
aws s3api put-bucket-tagging \
  --bucket terraform-refresh-demo-0f9a73bf \
  --tagging 'TagSet=[{Key=Name,Value=refresh-demo-bucket},{Key=Environment,Value=dev},{Key=ManagedBy,Value=terraform},{Key=AddedManually,Value=via-console}]'
```

### Step 3: Verify the tag was added in AWS

```bash
aws s3api get-bucket-tagging --bucket terraform-refresh-demo-0f9a73bf
```

Expected output — notice the new `AddedManually` tag:

```json
{
    "TagSet": [
        { "Key": "Environment",   "Value": "dev" },
        { "Key": "ManagedBy",     "Value": "terraform" },
        { "Key": "AddedManually", "Value": "via-console" },
        { "Key": "Name",          "Value": "refresh-demo-bucket" }
    ]
}
```

### Current situation after drift:

```
CONFIG (.tf)       STATE FILE          REAL AWS
──────────         ──────────          ──────────
3 tags             3 tags              4 tags  <-- AddedManually added here!
(no AddedManually) (no AddedManually)  (has AddedManually)
```

### Step 4: Save a backup of the state to reset between demos

```bash
cp terraform.tfstate terraform.tfstate.backup-demo
```

---

## 4. Demo 1: `terraform plan`

### What it does

1. **Refreshes** state from real world (in memory only — does NOT write to disk)
2. **Compares** real-world state against your `.tf` configuration
3. **Shows a full execution plan** of what changes it would make to bring the
   real world in line with your configuration

### Run it

```bash
terraform plan
```

### Expected Output

```
random_id.suffix: Refreshing state... [id=D5pzvw]
aws_s3_bucket.demo: Refreshing state... [id=terraform-refresh-demo-0f9a73bf]

Terraform will perform the following actions:

  # aws_s3_bucket.demo will be updated in-place
  ~ resource "aws_s3_bucket" "demo" {
        id   = "terraform-refresh-demo-0f9a73bf"
      ~ tags = {
          - "AddedManually" = "via-console" -> null    # <-- WILL REMOVE THIS
            "Environment"   = "dev"
            "ManagedBy"     = "terraform"
            "Name"          = "refresh-demo-bucket"
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

### Verify state was NOT modified

```bash
diff terraform.tfstate terraform.tfstate.backup-demo
```

Output: **no differences** — state file is untouched.

### Key Takeaways

- `terraform plan` detected the drift (AddedManually tag exists in AWS but not in config)
- It proposes to **REMOVE** the AddedManually tag to bring AWS back in line with your config
- **State file was NOT updated** — plan is read-only
- The refresh happened in-memory only

### The Logic Flow

```
Step 1: Refresh → Read AWS → found 4 tags in real world (in memory)
Step 2: Compare in-memory state vs config (.tf) → config says 3 tags
Step 3: Plan → "I need to remove AddedManually to match config"
Step 4: State file on disk → UNCHANGED
```

---

## 5. Demo 2: `terraform refresh`

> **WARNING: This command is DEPRECATED as of Terraform v0.15.4.**
> Use `terraform apply -refresh-only` instead (Demo 4).

### What it does

1. **Reads** the real world from the provider (AWS)
2. **Immediately writes** the real-world state into your state file — **NO CONFIRMATION!**
3. Does **NOT** plan or apply any infrastructure changes
4. Does **NOT** compare against your config

### Reset state first (if running demos in order)

```bash
cp terraform.tfstate.backup-demo terraform.tfstate
```

### Run it

```bash
terraform refresh
```

### Expected Output

```
random_id.suffix: Refreshing state... [id=D5pzvw]
aws_s3_bucket.demo: Refreshing state... [id=terraform-refresh-demo-0f9a73bf]

Outputs:

bucket_arn = "arn:aws:s3:::terraform-refresh-demo-0f9a73bf"
bucket_name = "terraform-refresh-demo-0f9a73bf"
```

Notice: **No plan, no confirmation, no diff shown.** It just... did it.

### Verify state WAS modified

```bash
diff terraform.tfstate terraform.tfstate.backup-demo
```

Output — state changed:

```
< "serial": 4,          (was 3)
< "AddedManually": "via-console",    <-- NOW IN STATE FILE!
```

Or check directly:

```bash
grep "AddedManually" terraform.tfstate
```

Output:

```
"AddedManually": "via-console",
"AddedManually": "via-console",
```

### After `terraform refresh`, the situation is:

```
CONFIG (.tf)       STATE FILE             REAL AWS
──────────         ──────────             ──────────
3 tags             4 tags  <-- UPDATED!   4 tags
(no AddedManually) (has AddedManually)    (has AddedManually)
```

### Now if you run `terraform plan`:

```bash
terraform plan
```

It will STILL show: **"remove AddedManually"** — because your config only has 3 tags.
The difference is that this time, terraform already knows from state that the tag exists,
so it doesn't need to discover it from AWS.

### Key Takeaways

- `terraform refresh` **silently updated the state file** with no confirmation
- It did **NOT** show you what changed (no diff)
- It did **NOT** touch your actual infrastructure
- **This is why it's deprecated** — it's dangerous because:
  - No preview of what will change in state
  - No approval step
  - If the real resource was deleted, it silently removes it from state
  - You could lose state data with no way to review first

### The Logic Flow

```
Step 1: Read AWS → found 4 tags
Step 2: Write to terraform.tfstate → DONE (serial incremented 3 → 4)
Step 3: (nothing else — no plan, no apply)
```

---

## 6. Demo 3: `terraform plan -refresh-only`

### What it does

1. **Reads** the real world from the provider (AWS)
2. **Compares** real world vs state file
3. **Shows you the drift** (what changed outside Terraform)
4. Does **NOT** plan any infrastructure changes
5. Does **NOT** update the state file

This is the **safe preview** version — it shows you drift without doing anything about it.

### Reset state first

```bash
cp terraform.tfstate.backup-demo terraform.tfstate
```

### Run it

```bash
terraform plan -refresh-only
```

### Expected Output

```
random_id.suffix: Refreshing state... [id=D5pzvw]
aws_s3_bucket.demo: Refreshing state... [id=terraform-refresh-demo-0f9a73bf]

Note: Objects have changed outside of Terraform

Terraform detected the following changes made outside of Terraform since the
last "terraform apply":

  # aws_s3_bucket.demo has changed
  ~ resource "aws_s3_bucket" "demo" {
        id                          = "terraform-refresh-demo-0f9a73bf"
      ~ tags                        = {
          + "AddedManually" = "via-console"    # <-- SHOWS what drifted
            "Environment"   = "dev"
            "ManagedBy"     = "terraform"
            "Name"          = "refresh-demo-bucket"
        }
      ~ tags_all                    = {
          + "AddedManually" = "via-console"
            # (3 unchanged elements hidden)
        }
        # (13 unchanged attributes hidden)
    }

This is a refresh-only plan, so Terraform will not take any actions to undo
these. If you were expecting these changes then you can apply this plan to
record the updated values in the Terraform state without changing any remote
objects.
```

### Verify state was NOT modified

```bash
diff terraform.tfstate terraform.tfstate.backup-demo
```

Output: **no differences** — state is untouched.

### Key Takeaways

- Shows you **exactly what drifted** (with a nice diff)
- The `+` sign shows something was ADDED in real world that state doesn't know about
- **Does NOT update state** — purely informational
- **Does NOT plan infra changes** — it won't suggest removing the tag
- This is the **safe investigation tool**: "What changed outside Terraform?"

### How it differs from `terraform plan`

| Aspect | `terraform plan` | `terraform plan -refresh-only` |
|---|---|---|
| Shows drift? | Indirectly (as part of the fix plan) | **Yes, explicitly** |
| Plans infra changes? | **Yes** (remove AddedManually) | **No** |
| Updates state? | No | No |
| Question it answers | "What will Terraform change?" | "What changed outside Terraform?" |

### The Logic Flow

```
Step 1: Read AWS → found 4 tags
Step 2: Compare AWS vs State → State has 3 tags, AWS has 4 → drift detected!
Step 3: Show drift diff → "AddedManually was added outside Terraform"
Step 4: State file on disk → UNCHANGED
Step 5: (no infrastructure plan — refresh-only stops here)
```

---

## 7. Demo 4: `terraform apply -refresh-only`

### What it does

1. **Reads** the real world from the provider (AWS)
2. **Compares** real world vs state file
3. **Shows you the drift** (same as `plan -refresh-only`)
4. **Asks for confirmation**
5. If approved, **writes the updated state to disk**
6. Does **NOT** change any actual infrastructure

This is the **safe replacement for `terraform refresh`** — it shows you what will
change in the state and asks before doing it.

### Reset state first

```bash
cp terraform.tfstate.backup-demo terraform.tfstate
```

### Run it

```bash
terraform apply -refresh-only
```

### Expected Output

```
random_id.suffix: Refreshing state... [id=D5pzvw]
aws_s3_bucket.demo: Refreshing state... [id=terraform-refresh-demo-0f9a73bf]

Note: Objects have changed outside of Terraform

Terraform detected the following changes made outside of Terraform since the
last "terraform apply":

  # aws_s3_bucket.demo has changed
  ~ resource "aws_s3_bucket" "demo" {
        id                          = "terraform-refresh-demo-0f9a73bf"
      ~ tags                        = {
          + "AddedManually" = "via-console"
            "Environment"   = "dev"
            "ManagedBy"     = "terraform"
            "Name"          = "refresh-demo-bucket"
        }
      ~ tags_all                    = {
          + "AddedManually" = "via-console"
            # (3 unchanged elements hidden)
        }
        # (13 unchanged attributes hidden)
    }

Would you like to update the Terraform state to reflect these detected changes?
  Terraform will write these changes to the state, but will NOT perform any
  actions to undo them.

  Enter a value: yes

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

### Verify state WAS modified

```bash
diff terraform.tfstate terraform.tfstate.backup-demo
```

Output — state was updated:

```
< "serial": 4,
< "AddedManually": "via-console",
```

### After `terraform apply -refresh-only`, the situation is:

```
CONFIG (.tf)       STATE FILE             REAL AWS
──────────         ──────────             ──────────
3 tags             4 tags  <-- UPDATED!   4 tags
(no AddedManually) (has AddedManually)    (has AddedManually)
```

Same result as `terraform refresh`, BUT:
- You **saw** exactly what was going to change in state
- You **confirmed** it before it happened
- You could have said `no` to abort

### Auto-approve variant (skip confirmation)

```bash
terraform apply -refresh-only -auto-approve
```

This behaves like the old `terraform refresh` but is the modern, supported way.

### Key Takeaways

- This is the **modern replacement for `terraform refresh`**
- Shows drift diff + asks for confirmation before writing state
- **Only updates state** — does NOT change real infrastructure
- After this, running `terraform plan` will still show "remove AddedManually"
  because your config still only defines 3 tags

---

## 8. Summary: When to Use What

### Quick Decision Guide

```
"I want to see what Terraform would do to my infrastructure"
  → terraform plan

"I want to check if something changed outside Terraform (drift detection)"
  → terraform plan -refresh-only

"I want to update my state file to match reality (accept drift into state)"
  → terraform apply -refresh-only

"I want to update state without any confirmation (old way, DON'T use)"
  → terraform refresh   ← DEPRECATED, avoid this
```

### Visual Flow Summary

```
                        ┌──────────────────────────────────┐
                        │        START: Drift Exists        │
                        │   (something changed outside TF)  │
                        └──────────────┬───────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
           ┌───────────────┐ ┌─────────────────┐ ┌─────────────────────┐
           │ terraform plan │ │ plan            │ │ apply               │
           │                │ │ -refresh-only   │ │ -refresh-only       │
           └───────┬───────┘ └────────┬────────┘ └──────────┬──────────┘
                   │                  │                     │
                   ▼                  ▼                     ▼
           ┌───────────────┐ ┌─────────────────┐ ┌─────────────────────┐
           │ Refreshes from│ │ Refreshes from  │ │ Refreshes from      │
           │ AWS (memory)  │ │ AWS (memory)    │ │ AWS (memory)        │
           └───────┬───────┘ └────────┬────────┘ └──────────┬──────────┘
                   │                  │                     │
                   ▼                  ▼                     ▼
           ┌───────────────┐ ┌─────────────────┐ ┌─────────────────────┐
           │ Compares      │ │ Shows state     │ │ Shows state drift   │
           │ config vs     │ │ drift only      │ │ + asks confirmation │
           │ real world    │ │ (what changed?) │ │                     │
           └───────┬───────┘ └────────┬────────┘ └──────────┬──────────┘
                   │                  │                     │
                   ▼                  ▼                     ▼
           ┌───────────────┐ ┌─────────────────┐ ┌─────────────────────┐
           │ Plans infra   │ │ State file:     │ │ WRITES updated      │
           │ changes to    │ │ UNCHANGED       │ │ state to disk       │
           │ match config  │ │                 │ │                     │
           │               │ │                 │ │ Infra: UNCHANGED    │
           │ State file:   │ │                 │ │                     │
           │ UNCHANGED     │ │                 │ │                     │
           └───────────────┘ └─────────────────┘ └─────────────────────┘
```

### Real-World Workflow (Recommended)

When you suspect drift:

```bash
# Step 1: Detect drift (safe, read-only)
terraform plan -refresh-only

# Step 2: If you WANT to accept the drift into state
terraform apply -refresh-only

# Step 3: If you want to bring infra back to match your config
terraform plan          # review the plan
terraform apply         # apply the changes

# Step 4: If you want to KEEP the manual change, update your config
#         Add the tag to main.tf, then run:
terraform plan          # should show "No changes"
```

---

## Cleanup

When done with the demo, remove the manually added tag:

```bash
# Option A: Let Terraform fix it (removes the manual tag)
terraform apply -auto-approve

# Option B: Or destroy everything
terraform destroy
```

Remove the backup file:

```bash
rm -f terraform.tfstate.backup-demo
```

---

## Bonus: Additional Useful Flags

```bash
# Skip the refresh step entirely (uses stale state as-is)
terraform plan -refresh=false

# Save plan to a file for later apply
terraform plan -out=myplan.tfplan
terraform apply myplan.tfplan

# Show detailed state for a resource
terraform state show aws_s3_bucket.demo

# List all resources in state
terraform state list
```
