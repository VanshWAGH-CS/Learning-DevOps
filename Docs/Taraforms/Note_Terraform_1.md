# 🏗️ Terraform — Infrastructure as Code (IaC) Notes

> **Author:** Vansh  
> **Topic:** Terraform Basics — What it is, Why it matters, How it compares to Ansible, and Hands-on Commands  

---

## 📌 Table of Contents

1. [What is Infrastructure as Code (IaC)?](#1-what-is-infrastructure-as-code-iac)
2. [What is Terraform?](#2-what-is-terraform)
3. [Why Does Terraform Matter?](#3-why-does-terraform-matter)
4. [Terraform vs Ansible](#4-terraform-vs-ansible)
5. [How Terraform Works — The Architecture](#5-how-terraform-works--the-architecture)
6. [Core Terraform Workflow](#6-core-terraform-workflow)
7. [Terraform Commands — Deep Dive](#7-terraform-commands--deep-dive)
8. [HCL — HashiCorp Configuration Language](#8-hcl--hashicorp-configuration-language)
9. [Hands-on Lab Walkthrough](#9-hands-on-lab-walkthrough)
10. [Key Concepts Glossary](#10-key-concepts-glossary)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. What is Infrastructure as Code (IaC)?

**Infrastructure as Code (IaC)** is the practice of managing and provisioning computing infrastructure (servers, networks, databases, etc.) through **machine-readable configuration files** instead of manual processes or interactive configuration tools.

### The Problem Before IaC

```
Old Way (Manual / ClickOps):
  Developer  →  Login to AWS Console  →  Click buttons  →  Server created
                     ↓
          ❌ Hard to repeat
          ❌ No version history
          ❌ Human error prone
          ❌ Not scalable
```

### The IaC Way

```
IaC Way:
  Developer  →  Write config file  →  Run tool  →  Infrastructure created
                     ↓
          ✅ Repeatable
          ✅ Version-controlled (Git)
          ✅ Automated
          ✅ Scalable
```

### IaC Benefits at a Glance

| Benefit | Explanation |
|---|---|
| **Consistency** | Same code = same infrastructure every time |
| **Version Control** | Track changes via Git like application code |
| **Automation** | No manual clicking; pipelines can run it |
| **Speed** | Provision hundreds of resources in minutes |
| **Documentation** | The code *is* the documentation |
| **Disaster Recovery** | Rebuild everything from code if infra is destroyed |

---

## 2. What is Terraform?

**Terraform** is an open-source Infrastructure as Code tool created by **HashiCorp**. It lets you define cloud infrastructure in a simple, human-readable language called **HCL (HashiCorp Configuration Language)** and then automatically creates, updates, or destroys that infrastructure.

```
┌─────────────────────────────────────────────────────────┐
│                      TERRAFORM                          │
│                                                         │
│   You write .tf files  →  Terraform figures out        │
│   what needs to change  →  Makes it happen             │
│                                                         │
│   Works with: AWS, Azure, GCP, Kubernetes,             │
│               GitHub, Docker, and 1000+ providers      │
└─────────────────────────────────────────────────────────┘
```

### Terraform is Declarative

You describe the **desired end state**, not the steps to get there.

```hcl
# You say WHAT you want:
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Terraform figures out HOW to create it
```

---

## 3. Why Does Terraform Matter?

### Real-World Scenario

Imagine your company runs 50 EC2 servers on AWS. Without Terraform:
- Someone needs to manually click through the console
- If a server dies, recreating it exactly is hard
- Spinning up a dev/staging/prod copy is a week of work

With Terraform:
```bash
terraform apply   # Entire environment up in minutes
terraform destroy # Tear it all down cleanly
# Copy the folder, change a variable → new environment
```

### Terraform in a DevOps Pipeline

```
Developer pushes code
        ↓
  CI/CD Pipeline runs
        ↓
  terraform plan   ← Shows what will change
        ↓
  Code Review / Approval
        ↓
  terraform apply  ← Changes are made to real infra
        ↓
  Infrastructure Updated ✅
```

---

## 4. Terraform vs Ansible

This is one of the most common interview questions. They are **not competitors** — they solve different problems.

### Key Difference: Provisioning vs Configuration

```
┌──────────────────────────────────────────────────────────────┐
│  TERRAFORM                      ANSIBLE                      │
│  "Build the house"              "Furnish and maintain it"    │
│                                                              │
│  • Creates infra                • Configures software        │
│  • EC2, VPCs, DBs               • Installs nginx, MySQL      │
│  • Declarative                  • Procedural (steps)         │
│  • Stateful (tracks state)      • Stateless (runs top-down)  │
│  • Better for cloud             • Better for OS/app config   │
└──────────────────────────────────────────────────────────────┘
```

### Detailed Comparison Table

| Feature | Terraform | Ansible |
|---|---|---|
| **Primary Use** | Provisioning infrastructure | Configuration management |
| **Language** | HCL (declarative) | YAML Playbooks (procedural) |
| **State Management** | Yes — maintains a `terraform.tfstate` file | No — stateless |
| **Idempotency** | Built-in (knows current state) | Achieved via careful playbook writing |
| **Cloud Resources** | Excellent | Limited (uses cloud modules) |
| **Software Config** | Limited | Excellent |
| **Agentless?** | Yes | Yes |
| **When to use** | Create/destroy infra | Install packages, edit configs |

### They Work Best Together

```
Step 1: Terraform provisions the VM on AWS
            ↓
Step 2: Ansible connects to the VM and installs nginx + app
            ↓
Step 3: Application is running ✅
```

---

## 5. How Terraform Works — The Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    TERRAFORM ARCHITECTURE                       │
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌───────────────┐   │
│  │  .tf Files   │────▶│  Terraform   │────▶│   Provider    │   │
│  │  (Your code) │     │    Core      │     │  (AWS/Azure/  │   │
│  └──────────────┘     │              │     │   GCP/Local)  │   │
│                       │  • Parses    │     └───────────────┘   │
│  ┌──────────────┐     │  • Plans     │             │           │
│  │  .tfstate    │────▶│  • Applies   │             ▼           │
│  │  (State file)│     │              │     ┌───────────────┐   │
│  └──────────────┘     └──────────────┘     │  Real Infra   │   │
│                                            │  (Cloud/Local)│   │
│                                            └───────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

**1. Configuration Files (`.tf`)**  
Where you write your desired infrastructure in HCL.

**2. Providers**  
Plugins that let Terraform talk to different platforms (AWS, GCP, Azure, GitHub, Docker, `local`, etc.).

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}
```

**3. State File (`terraform.tfstate`)**  
A JSON file Terraform uses to track what it has already created. This is how Terraform knows whether to create, update, or delete a resource.

**4. Resources**  
The actual things Terraform manages — files, servers, databases, etc.

---

## 6. Core Terraform Workflow

```
  ┌──────────┐
  │  Write   │  Create .tf configuration files
  │  Config  │
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  init    │  Download providers & initialize
  │          │  terraform init
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  plan    │  Preview what will happen (dry run)
  │          │  terraform plan
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │  apply   │  Actually create/modify infrastructure
  │          │  terraform apply
  └────┬─────┘
       │
       ▼
  ┌──────────┐
  │ destroy  │  Tear down everything (when needed)
  │          │  terraform destroy
  └──────────┘
```

---

## 7. Terraform Commands — Deep Dive

### `terraform init`

**What it does:** Initializes your working directory. Downloads the required provider plugins.

```bash
terraform init
```

**Output explained from lab:**
```
- Finding hashicorp/local versions matching "~> 2.4"...
# Terraform looks for the 'local' provider matching version 2.4.x

- Installing hashicorp/local v2.9.0...
# Downloads the provider plugin

Terraform has created a lock file .terraform.lock.hcl
# Lock file pins exact versions so team members get the same provider
```

**Files created after init:**
```
terraform-files/
├── main.tf                  ← Your config (you wrote this)
├── .terraform/              ← Downloaded providers (auto-created)
│   └── providers/
│       └── hashicorp/local/
└── .terraform.lock.hcl      ← Version lock file (commit to Git!)
```

> 💡 **Run `init` when:** Starting a new project, adding a new provider, or cloning a repo.

---

### `terraform plan`

**What it does:** Creates an execution plan — a dry run that shows exactly what Terraform *will* do without actually doing it.

```bash
terraform plan
```

**Reading the plan output:**

```
+ create    ← This resource will be CREATED (green)
~ update    ← This resource will be UPDATED in-place
- destroy   ← This resource will be DESTROYED (red)
-/+ replace ← Destroy and recreate (breaking change)
```

**From the lab:**
```
# local_file.example will be created
+ resource "local_file" "example" {
    + content  = "Hello from Terraform on Ubuntu!"  ← Known now
    + filename = "hello.txt"                        ← Known now
    + id       = (known after apply)                ← Only known after creation
  }

Plan: 1 to add, 0 to change, 0 to destroy.
#     ↑ counts make it easy to review at a glance
```

> 💡 **Best practice:** Always run `plan` before `apply`. In CI/CD, save the plan with `terraform plan -out=tfplan` and then `terraform apply tfplan` to guarantee the exact plan is applied.

---

### `terraform apply`

**What it does:** Executes the plan and makes actual changes to your infrastructure.

```bash
terraform apply
```

Terraform will:
1. Show the plan again
2. Ask for confirmation (`yes` to proceed)
3. Create/modify/destroy resources
4. Update the state file

```bash
# From the lab:
Enter a value: yes

local_file.example: Creating...
local_file.example: Creation complete after 0s [id=10e216b3...]
#                                                  ↑ SHA1 hash of the file content

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Skip confirmation (use carefully in automation):**
```bash
terraform apply -auto-approve
```

> ⚠️ **Warning:** `apply` makes REAL changes. Always review the plan carefully first.

---

### `terraform destroy`

**What it does:** Destroys all infrastructure managed by the current Terraform config.

```bash
terraform destroy
```

```bash
# From the lab — Terraform shows exactly what it will delete:
- resource "local_file" "example" {
    - content  = "Hello from Terraform on Ubuntu!" -> null
    - filename = "hello.txt" -> null
  }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Only 'yes' will be accepted to confirm.

  Enter a value: no   ← Chose 'no' in the lab, destroy cancelled
Destroy cancelled.
```

**Destroy a specific resource only:**
```bash
terraform destroy -target=local_file.example
```

> 💡 Use `destroy` to clean up dev/test environments to save costs.

---

### Other Useful Commands

| Command | Purpose |
|---|---|
| `terraform fmt` | Auto-format your `.tf` files |
| `terraform validate` | Check config syntax without planning |
| `terraform show` | Display current state in human-readable form |
| `terraform state list` | List all resources in state |
| `terraform output` | Show output values |
| `terraform graph` | Generate dependency graph (pipe to Graphviz) |

---

## 8. HCL — HashiCorp Configuration Language

### Basic Structure

```hcl
# This is a comment

# BLOCK TYPE "RESOURCE TYPE" "LOCAL NAME" {
resource "local_file" "example" {
  # Arguments (key = value)
  content  = "Hello from Terraform on Ubuntu!"
  filename = "hello.txt"
}
```

### From the Lab — `main.tf` breakdown

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"  # Provider registry path
      version = "~> 2.4"          # ~> means >= 2.4.0 and < 3.0.0
    }
  }
}

resource "local_file" "example" {
#  ↑ block type    ↑ resource type   ↑ your chosen name
  content  = "Hello from Terraform on Ubuntu!"
  filename = "hello.txt"
  # This creates a file named hello.txt with the given content
}
```

### Resource Naming Convention

```
resource "PROVIDER_TYPE" "LOCAL_NAME"

"local_file"   → provider: local, type: file
"aws_instance" → provider: aws, type: instance
"google_storage_bucket" → provider: google, type: storage_bucket

LOCAL_NAME → Only used inside your Terraform code to reference this resource
             Not the actual name of the thing created
```

---

## 9. Hands-on Lab Walkthrough

Here's a full summary of what happened in the lab session:

```bash
# Step 1: Create project directory
mkdir terraform-files
cd terraform-files

# Step 2: Write the config
nano main.tf
# (wrote the local_file resource config)

# Step 3: Initialize — downloads 'local' provider
terraform init
# ✅ hashicorp/local v2.9.0 installed

# Step 4: Preview the plan
terraform plan
# ✅ Shows: 1 to add, 0 to change, 0 to destroy

# Step 5: Apply — actually creates hello.txt
terraform apply
# Entered 'yes' at prompt
# ✅ local_file.example: Creation complete

# Step 6: Verify the file was created
cat hello.txt
# Output: Hello from Terraform on Ubuntu!

# Step 7: Try destroy (cancelled with 'no')
terraform destroy
# Showed what would be deleted
# Entered 'no' → Destroy cancelled
```

### What happened under the hood

```
terraform apply
      │
      ▼
Terraform reads main.tf
      │
      ▼
Compares with terraform.tfstate (empty first time)
      │
      ▼
Calls the 'local' provider's "create file" API
      │
      ▼
hello.txt is created on disk
      │
      ▼
terraform.tfstate is updated with resource info
(content hashes, permissions, id, etc.)
```

---

## 10. Key Concepts Glossary

| Term | Meaning |
|---|---|
| **Provider** | Plugin that lets Terraform manage a specific platform (AWS, local, etc.) |
| **Resource** | A single piece of infrastructure Terraform manages |
| **State** | Terraform's record of what infrastructure currently exists |
| **Plan** | A preview of changes Terraform will make |
| **HCL** | HashiCorp Configuration Language — the language `.tf` files are written in |
| **Idempotent** | Running the same config multiple times produces the same result |
| **Declarative** | You describe the desired state; the tool figures out how to achieve it |
| **Lock file** | `.terraform.lock.hcl` pins exact provider versions for consistency |
| **`(known after apply)`** | Value that Terraform can't determine until the resource is actually created |

---

## 11. Quick Reference Cheat Sheet

```bash
# INITIALIZE
terraform init              # Download providers, set up directory

# PREVIEW
terraform plan              # Show what will change (safe, no changes made)
terraform plan -out=tfplan  # Save plan to file

# APPLY
terraform apply             # Apply changes (asks for confirmation)
terraform apply -auto-approve  # Skip confirmation (careful!)
terraform apply tfplan      # Apply a saved plan

# DESTROY
terraform destroy           # Destroy all managed resources
terraform destroy -target=RESOURCE  # Destroy specific resource

# INSPECT
terraform show              # Show current state
terraform state list        # List all resources
terraform output            # Show output values
terraform graph             # Dependency graph

# CODE QUALITY
terraform fmt               # Format code
terraform validate          # Validate syntax
```

---

## Further Learning

- [Terraform Official Docs](https://developer.hashicorp.com/terraform/docs)
- [Terraform Registry (Providers & Modules)](https://registry.terraform.io/)
- [Learn Terraform (HashiCorp Tutorials)](https://developer.hashicorp.com/terraform/tutorials)
- Next steps: Try provisioning an actual AWS EC2 instance with `hashicorp/aws` provider

---

*Notes based on hands-on lab practice on Ubuntu with Terraform + local provider.*
