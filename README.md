# Terraform Guidelines

[![Latest Release](https://img.shields.io/github/v/tag/jameswoolfenden/terraform-guidelines.svg)](https://github.com/jameswoolfenden/terraform-guidelines)
[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white)](https://github.com/pre-commit/pre-commit)

By [James Woolfenden](https://www.linkedin.com/in/jameswoolfenden/)

## A style guide for writing Terraform

This is a guide to writing Terraform, it follows the Hashicorp guide to creating modules for the [Terraform Registry](https://www.terraform.io/docs/registry/modules/publish.html) and their [standard structure](https://www.terraform.io/docs/modules/index.html#standard-module-structure).

There are many successful ways of writing your tf, this one is tried and field tested.

---

## Naming

Use lower-case names for resources. Use `_` as a separator. Names must be self-explanatory — several lower-case words if needed.

Use descriptive and non-environment-specific names to identify resources.

Avoid tautologies — the resource type already tells you what it is:

```terraform
# bad
resource "aws_iam_policy" "ec2_policy" { ... }

# good
resource "aws_iam_policy" "ec2_read_only" { ... }
```

---

## Hard-coding

Don't hard-code values in resources. Add variables and set defaults.

You can avoid limiting yourself with policies and resources by making resources optional or over-ridable.

```terraform
resource "aws_iam_role" "codebuild" {
  name  = "codebuildrole-${var.name}"
  count = var.role == "" ? 1 : 0

  assume_role_policy = data.aws_iam_policy_document.codebuild_assume.json

  tags = var.common_tags
}

data "aws_iam_policy_document" "codebuild_assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["codebuild.amazonaws.com"]
    }
  }
}
```

Avoid HEREDOCs for IAM policies — use **data.aws_iam_policy_document** objects instead, as shown above. They are type-safe, composable, and diff cleanly.

---

## Locals

Use `locals` for values that are computed from other inputs or repeated across the configuration. Locals reduce duplication and make intent clear.

```terraform
locals {
  bucket_name = "${var.project}-${var.environment}-state"
  common_tags = merge(var.common_tags, {
    managed_by  = "terraform"
    project     = var.project
    environment = var.environment
  })
}
```

Rules for locals:
- Prefer locals over repeating the same expression in multiple places
- Do not use locals as a substitute for variables — locals are not user-configurable
- Name locals as clearly as you would name a variable
- Avoid deeply nested locals that are hard to trace

---

## `count` vs `for_each`

Prefer `for_each` over `count` in almost all cases.

`count` is appropriate only for simple on/off flags:

```terraform
resource "aws_cloudwatch_log_group" "this" {
  count = var.enable_logging ? 1 : 0
  name  = "/aws/lambda/${var.function_name}"
}
```

Use `for_each` when creating multiple instances of a resource. It uses a map or set, giving each instance a stable identity. Adding or removing one item does not affect the others — unlike `count`, which uses integer indices and causes cascading destroys when the list changes.

```terraform
variable "buckets" {
  type    = set(string)
  default = ["assets", "logs", "backups"]
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "${var.project}-${each.key}"
  tags     = var.common_tags
}
```

With a map, each instance gets a meaningful key:

```terraform
resource "aws_iam_user" "this" {
  for_each = { for u in var.users : u.name => u }
  name     = each.key
  tags     = each.value.tags
}
```

Never use `count` to iterate a list of objects — any insertion or deletion shifts indices and triggers unexpected destroys.

---

## `lifecycle` blocks

Use `lifecycle` to protect critical resources and control how Terraform manages updates.

### Prevent accidental deletion

```terraform
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
```

### Create before destroy

Use this when a resource must exist before its predecessor is removed, such as when a replacement is needed without downtime:

```terraform
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```

### Ignore external changes

Use `ignore_changes` sparingly — only when an external process legitimately modifies an attribute that Terraform should not revert:

```terraform
resource "aws_autoscaling_group" "app" {
  # ...
  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

### replace_triggered_by

Force replacement of a resource when another resource or value changes:

```terraform
resource "aws_instance" "web" {
  # ...
  lifecycle {
    replace_triggered_by = [aws_launch_template.web.id]
  }
}
```

---

## Sensitive values

Mark variables and outputs as `sensitive` when they contain secrets. Terraform will redact them from plan and apply output.

```terraform
variable "database_password" {
  type        = string
  description = "Master password for the RDS instance"
  sensitive   = true
}

output "connection_string" {
  description = "Database connection string"
  value       = "postgres://admin:${var.database_password}@${aws_db_instance.main.endpoint}/app"
  sensitive   = true
}
```

Sensitive values still appear in state — always encrypt your state backend at rest.

---

## Dynamic blocks

Use dynamic blocks when the number of nested configuration blocks is driven by a variable. Avoid using them when a fixed block would be clearer.

```terraform
resource "aws_security_group" "app" {
  name   = "${var.name}-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = var.common_tags
}
```

The variable:

```terraform
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = []
}
```

---

## `moved` blocks

When you rename a resource or move it into a module, use a `moved` block instead of destroying and recreating it. This is available from Terraform 1.1+.

```terraform
moved {
  from = aws_s3_bucket.logs
  to   = aws_s3_bucket.audit_logs
}
```

For moving into a module:

```terraform
moved {
  from = aws_iam_role.codebuild
  to   = module.codebuild.aws_iam_role.this
}
```

Remove the `moved` block after the state has been updated and the change is applied across all environments.

---

## `import` blocks

From Terraform 1.5+, use declarative `import` blocks rather than the imperative `terraform import` command. This makes imports reviewable, repeatable, and part of your plan.

```terraform
import {
  to = aws_s3_bucket.existing_logs
  id = "my-existing-logs-bucket"
}

resource "aws_s3_bucket" "existing_logs" {
  bucket = "my-existing-logs-bucket"
  tags   = var.common_tags
}
```

Run `terraform plan` to verify the import before applying. From Terraform 1.6+, use `terraform generate-config-out` to scaffold the resource configuration automatically.

---

## `check` blocks

From Terraform 1.5+, `check` blocks let you assert postconditions about your infrastructure without failing the apply. They surface as warnings.

```terraform
check "alb_healthy" {
  data "aws_lb_target_group" "this" {
    arn = aws_lb_target_group.app.arn
  }

  assert {
    condition     = data.aws_lb_target_group.this.load_balancing_algorithm_type != null
    error_message = "Target group is not attached to a load balancer."
  }
}
```

Use `precondition` and `postcondition` inside resources and data sources when the check must block the apply:

```terraform
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "Only x86_64 AMIs are supported."
    }
  }
}
```

---

## Templates

This is the Terraform code that is environment specific. If your Templates are application specific the code should live with the code that requires it, create a folder in the root of the repository and call it **IAC**, similar to this for the repository aws-lexbot-handlers:

```bash
23043-5510:/mnt/c/aws-lexbot-handler# ls -l
total 64
-rwxrwxrwx    1 jimw     jimw          1719 Mar  7 11:02 README.MD
-rwxrwxrwx    1 jimw     jimw           411 Mar  7 11:02 buildno.sh
-rwxrwxrwx    1 jimw     jimw          1136 Mar 18 15:43 buildspec.yml
-rwxrwxrwx    1 jimw     jimw           489 Mar 18 15:40 getlatest.ps1
-rwxrwxrwx    1 jimw     jimw           479 Mar  7 11:02 getlatest.sh
drwxrwxrwx    1 jimw     jimw           512 Feb 26 16:22 iac
drwxrwxrwx    1 1000     1000           512 Mar 18 15:41 node_modules
-rwxrwxrwx    1 jimw     jimw         49979 Mar 18 15:41 package-lock.json
-rwxrwxrwx    1 jimw     jimw          1579 Mar 18 15:41 package.json
drwxrwxrwx    1 jimw     jimw           512 Feb  5 11:51 powershell
-rwxrwxrwx    1 jimw     jimw           147 Mar  7 11:02 setlatest.sh
```

Inside the **iac** folder, break down the **templates** used:

```bash
total 0
drwxrwxrwx    1 jimw     jimw           512 May 28 11:22 codebuild
drwxrwxrwx    1 jimw     jimw           512 Apr  2 11:00 repository
```

This example has an AWS CodeCommit repository (self describing) and an AWS Codebuild, that has multiple environments:

```bash
total 0
drwxrwxrwx    1 jimw     jimw           512 Apr 24 23:28 dev
drwxrwxrwx    1 jimw     jimw           512 Apr 24 23:29 prod
```

Inside each of these folders is an environment-specific template:

```bash
total 19
-rwxrwxrwx    1 jimw     jimw           800 May 28 11:21 Makefile
-rwxrwxrwx    1 jimw     jimw          1324 Mar  7 11:02 README.md
-rwxrwxrwx    1 jimw     jimw           709 Mar  7 11:02 aws_iam_policy.additionalneeds.tf
-rwxrwxrwx    1 jimw     jimw            40 Mar  7 11:02 data.aws_current.region.current.tf
-rwxrwxrwx    1 jimw     jimw           579 May 28 11:23 module.codebuild.tf
-rwxrwxrwx    1 jimw     jimw           239 Mar  7 11:02 outputs.tf
-rwxrwxrwx    1 jimw     jimw           208 Apr 24 23:30 provider.tf
-rwxrwxrwx    1 jimw     jimw           349 Mar  7 11:02 remote_state.tf
-rwxrwxrwx    1 jimw     jimw          1531 May 28 11:25 terraform.tfvars
-rwxrwxrwx    1 jimw     jimw           618 May 28 11:20 variables.tf
```

There's a lot of files here and some repetition - that violates DRY principles, but with IaC, favour being explicit.
Each template is directly runnable using the Terraform CLI with no wrapper script required.
Use a generator like [tf-scaffold](https://github.com/JamesWoolfenden/tf-scaffold) to automate template creation.

Tf-Scaffold creates:

### .gitignore

Has good defaults for working with Terraform.

```yaml
terraform.tfplan
terraform.tfstate
.terraform/
./*.tfstate
*.backup
*.DS_Store
*.log
*.bak
*~
.*.swp
.project
```

**Important:** Do **not** add `.terraform.lock.hcl` to `.gitignore`. This file records the exact provider versions resolved during `terraform init` and should be committed to version control so all team members and CI use identical provider binaries.

### .github/workflows/main.yml

If you're using GitHub you can add a basic CI workflow.

This is a sample workflow for the minimum validation of a module **main.yml**:

```yml
---
name: Verify and Bump
on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - master

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~> 1.0"
      - run: terraform init
        working-directory: example/examplea
      - run: terraform validate
        working-directory: example/examplea
      - name: Test with Checkov
        run: |
          pip install checkov
          checkov -d .

  version:
    name: versioning
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - name: Bump version and push tag
        uses: anothrNick/github-tag-action@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DEFAULT_BUMP: patch
          WITH_V: "true"
    needs: build
```

### .terraformignore

If using Terraform Cloud, the files not to upload to their cloud.

```ini
.terraform/
*.exe
*.tfstate
*.backup
*.bak
*.info
```

### .pre-commit-config.yaml

Has a standard set of pre-commit hooks for working with Terraform and AWS. You'll need to install the pre-commit framework https://pre-commit.com/#install.
You can modify the default behaviour of Git on a workstation to always implement hooks:

On Windows (Pwsh)

```powershell
pre-commit init-templatedir $HOME\.git-template
git config --global init.templateDir $HOME\.git-template
```

On Nix

```bash
git config --global init.templateDir ~/.git-template
pre-commit init-templatedir ~/.git-template
```

When you clone a repo with hooks configured they are now installed automatically.

Add this file (or a similar one) to any new repository as **pre-commit-config.yaml** in the root:

```yaml
---
# yamllint disable rule:line-length
default_language_version:
  python: python3
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-json
      - id: check-merge-conflict
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: pretty-format-json
        args:
          - --autofix
      - id: detect-aws-credentials
      - id: detect-private-key
  - repo: https://github.com/Lucas-C/pre-commit-hooks
    rev: v1.5.5
    hooks:
      - id: forbid-tabs
        exclude_types:
          - python
          - javascript
          - dtd
          - markdown
          - makefile
          - xml
        exclude: binary|\.bin$
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.2
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: checkov
```

Use hooks that don't clash and ensure they are cross-platform compatible.

``` bash
pre-commit install
```

Hooks choices are a matter for each project, but if you are using Terraform or AWS the credentials and private key hooks or equivalent are required.

### main.tf

This is an expected file for Terraform modules. Remove it if this is a template and add a **module.tf**.

### Makefile

A helper file to make life easier. Problematic if you are cross-platform as make isn't very good at that. If you use Windows, update the PowerShell profile with equivalent helper functions instead.

```makefile
#Makefile
ifdef OS
   BLAT = $(powershell  -noprofile rm .\.terraform\ -force -recurse)
else
   ifeq ($(shell uname), Linux)
      RM  = rm .terraform/modules/ -fr
      BLAT= rm .terraform/ -fr
   endif
endif

.PHONY: all

all: init plan build

init:
   $(RM)
   terraform init -reconfigure

plan:
   terraform plan -refresh=true

p:
   terraform plan -refresh=true

apply: build

build:
   terraform apply -auto-approve

check: init
   terraform plan -detailed-exitcode

destroy: init
   terraform destroy

docs:
   terraform-docs md . > README.md

valid:
   terraform fmt -check=true -diff=true
   checkov -d . --external-checks-dir ../../checkov

target:
   @read -p "Enter Module to target:" MODULE;
   terraform apply -target $$MODULE

purge:
   $(BLAT)
   terraform init -reconfigure
```

### outputs.tf

A standard place to return values, either to the screen or to pass back from a module. Every output must have a `description`. Mark outputs that expose secrets as `sensitive = true`.

```terraform
output "bucket_arn" {
  description = "ARN of the S3 state bucket"
  value       = aws_s3_bucket.state.arn
}

output "kms_key_arn" {
  description = "ARN of the KMS key used for encryption"
  value       = aws_kms_key.state.arn
  sensitive   = false
}
```

In a root module (a deployed template), outputs are visible in the console and available via `terraform output`. In a child module, outputs are the only values the calling module can access — design them as a deliberate API.

### provider.aws.tf

Or whatever your provider is.
Required. The most basic provider block for AWS lives in **terraform.tf** alongside `required_providers`:

```terraform
terraform {
  required_version = "~> 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

### README.md

Where all the information goes. Use [terraform-docs](https://github.com/terraform-docs/terraform-docs) to auto-generate the inputs/outputs table.

### example.auto.tfvars

Files ending `.auto.tfvars` are picked up automatically by Terraform locally and in Terraform Cloud.
This is the standard file for setting your variables in.

Do not commit `.auto.tfvars` files that contain real secrets — use environment variables or a secrets manager instead.

### variables.tf

For defining your variables and setting default values. Each variable must define its `type` and have an adequate `description`.
Also contains a map variable `common_tags`, which should be extended and used on every taggable object.

As of Terraform 0.13, custom input validation is available:

```terraform
variable "config_file" {
  type        = string
  description = "Path to the configuration file"
  validation {
    condition     = length(var.config_file) > 0
    error_message = "This value cannot be an empty string."
  }
}
```

#### Check to see item is set to valid options only

```terraform
variable "log_level" {
  type        = string
  description = "Log level. Must be one of DEBUG, INFO, WARNING, ERROR, CRITICAL."

  validation {
    condition     = contains(["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"], var.log_level)
    error_message = "The value must be one of DEBUG, INFO, WARNING, ERROR, CRITICAL."
  }
}
```

#### Constrain format

```terraform
variable "tracing_mode" {
  type        = string
  description = "X-Ray tracing setting"
  default     = "Active"

  validation {
    condition     = contains(["PassThrough", "Active"], var.tracing_mode)
    error_message = "Tracing mode can only be PassThrough or Active."
  }
}
```

#### Check prefix or format

```terraform
variable "runtime" {
  type        = string
  description = "Lambda runtime"
  default     = "python3.12"
  validation {
    condition     = startswith(var.runtime, "python")
    error_message = "This module uses Python — runtime must start with \"python\"."
  }
}
```

#### Optional object attributes (Terraform 1.3+)

Use `optional()` inside an object type to allow callers to omit fields and receive a default instead:

```terraform
variable "log_config" {
  type = object({
    enabled   = optional(bool, true)
    retention = optional(number, 30)
    level     = optional(string, "INFO")
  })
  default     = {}
  description = "Log configuration. All fields are optional."
}
```

### .github/dependabot.yml

Sets the repository to be automatically dependency scanned in GitHub.

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "terraform"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## Modules

You've written some TF and you're about to duplicate its functionality — it's time to abstract to a module. A module should be more than just one resource, it should add something.
Modules should be treated like application services with a separate code repository for each module.

Each module should have at least one example included that demonstrates its usage. This example can be used as a test for that module, here it's called **examplea**.

```bash
examples/examplea
```

See <https://github.com/JamesWoolfenden/terraform-aws-codecommit> for a conformant example.

Use lowercase for all folder names, avoid spaces.

### Module design principles

- A module should encapsulate a meaningful unit of infrastructure — not just a single resource
- Accept all provider configuration from the caller; never hardcode provider settings inside a module
- Expose everything the caller might need via outputs — treat outputs as a public API
- Use `variable` descriptions to document what each input does
- Avoid data sources that reach outside the module's scope; pass values in as variables instead

---

## Files

### Name your files after their contents

For a security group called "elastic", the resource is `aws_security_group.elastic`, so the file is **aws_security_group.elastic.tf**. Be explicit. It will save you time.

### Comments

Use Markdown for documentation. Many fmt and parser tools break on in-line HCL comments. Make sure to add descriptions to your variables and outputs — that is where user-facing documentation belongs.

### One resource per file

**Exception**: Group resources where it genuinely makes logical sense — security group with its rules, a route table with its routes.

This follows the standard practice in most development languages. It helps you navigate code in your editor and keeps diffs focused on a single concern.

---

## Be Specific

You have 2 choices with dependencies: live on the bleeding edge, or fix your versions. Fix them.

### Fix the version of Terraform you use

The whole team needs to use the same version of the tool until you decide as a team to update.
Create a file called **terraform.tf** in your template:

```terraform
terraform {
  required_version = "~> 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Do a coordinated update of provider versions and tool versions at regular intervals — e.g. every quarter.

### Commit `.terraform.lock.hcl`

After running `terraform init`, Terraform writes a lock file recording the exact provider versions and checksums selected. **Commit this file.** It guarantees every developer and every CI run uses exactly the same provider binaries.

```bash
git add .terraform.lock.hcl
git commit -m "chore: pin provider versions"
```

To update the lock file intentionally:

```bash
terraform init -upgrade
```

### Fix the version of the modules you consume

In your **module.tf** file, set the version of the module. If you author modules, tag successful builds.

```terraform
module "codebuild" {
  source      = "jameswoolfenden/codebuild/aws"
  version     = "0.1.41"
  root        = var.root
  description = var.description
}
```

If using a git source reference, pin to a tag:

```terraform
module "vpc" {
  source = "git::https://github.com/example/terraform-aws-vpc.git?ref=v2.3.0"
}
```

### Fix the version of the providers you use

Specify the version in your **terraform.tf** file using `required_providers`. The `version` argument inside `provider` blocks is deprecated since Terraform 0.13 — do not use it.

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

---

## State

By default, Terraform stores state locally in `terraform.tfstate`. Local state does not scale — it lives only on one machine, has no locking, and contains secrets in plaintext.

Never commit state files to version control.

Use [remote state](https://www.terraform.io/docs/state/remote.html) from day one:

- [S3 + DynamoDB](https://www.terraform.io/docs/backends/types/s3.html) — the standard choice for AWS; S3 for storage, DynamoDB for locking. A ready-made module: [terraform-aws-statebucket](https://registry.terraform.io/modules/JamesWoolfenden/statebucket/aws)
- [GCS](https://www.terraform.io/docs/backends/types/gcs.html) — for GCP; object versioning provides locking
- [Azure Blob Storage](https://www.terraform.io/docs/backends/types/azurerm.html) — native locking and consistency checking
- [Terraform Cloud/Enterprise](https://www.terraform.io/docs/backends/types/remote.html) — provider-agnostic; good for multi-cloud teams

Enable [state locking](https://www.terraform.io/docs/state/locking.html) wherever your backend supports it — it prevents concurrent applies from corrupting state.

### State manipulation

Avoid manual state manipulation. When you do need it, treat it as a controlled operation:

- Use `terraform state list` to inspect what Terraform tracks
- Use `moved` blocks (Terraform 1.1+) rather than `terraform state mv` — they are reviewable and repeatable
- Use `import` blocks (Terraform 1.5+) rather than `terraform import` — same reason
- Take a state backup before any manual operation: `terraform state pull > backup.tfstate`

---

## Environments

Do not use Terraform workspaces for environment separation — they share a backend configuration and require conditional logic that makes code hard to read and reason about.

Prefer one of these patterns instead:

### Directory per environment

```bash
environments/
  dev/
    terraform.tf
    main.tf
    terraform.tfvars
  staging/
    terraform.tf
    main.tf
    terraform.tfvars
  prod/
    terraform.tf
    main.tf
    terraform.tfvars
```

Each directory has its own state file and backend configuration. Applying to `prod` can never accidentally affect `dev`.

### Shared module, environment-specific tfvars

A single root module is invoked with a different var file per environment:

```bash
terraform init -backend-config=env/prod/backend.hcl
terraform apply -var-file=env/prod/terraform.tfvars
```

This pattern reduces duplication while keeping environments isolated at the state level.

---

## Layout

Mandate the use of the standard pre-commit hooks — this enforces `terraform fmt` on every Git commit. End of problem.

Recommended file layout for a module:

```bash
terraform-aws-example/
  main.tf               # primary resources
  variables.tf          # input variable declarations
  outputs.tf            # output declarations
  terraform.tf          # required_version + required_providers
  locals.tf             # local value declarations (if needed)
  data.tf               # data source declarations (if needed)
  README.md             # generated by terraform-docs
  .terraform.lock.hcl   # committed provider lock file
  examples/
    examplea/
      main.tf
      outputs.tf
      variables.tf
      example.auto.tfvars
  tests/
    examplea.tftest.hcl
```

---

## Protecting Secrets

Protect your secrets by installing using the pre-commit file and the hooks from the standard set:

```yaml
- id: detect-aws-credentials
- id: detect-private-key
```

Other options include using git-secrets or Talisman. Use and mandate one, for all. Don't be that person.

In addition:
- Mark sensitive variables and outputs with `sensitive = true`
- Store secrets in a secrets manager (AWS Secrets Manager, Vault, Azure Key Vault) and reference them via data sources
- Never pass secrets as command-line `-var` arguments — they appear in shell history and CI logs
- Ensure your state backend encrypts at rest — all secrets end up in state

---

## Security Review

The checks below serve two purposes: guidance for authors writing Terraform, and a structured checklist for automated or manual security review. Each check includes a severity rating, the pattern to look for, a bad example, a good example, and remediation advice.

Severity scale: **Critical** — fix before merge | **High** — fix in current sprint | **Medium** — schedule | **Low** — best practice

---

### Secrets and Credentials

#### No hardcoded credentials

**Severity:** Critical

Look for AWS access keys, passwords, tokens, or private key material as literal strings in any `.tf` file.

```terraform
# bad
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# good — credentials sourced from environment or instance profile
provider "aws" {
  region = var.region
}
```

**Remediation:** Remove credentials from code. Use instance profiles, IRSA, or environment variables. Rotate any exposed credentials immediately.

---

#### Sensitive variables and outputs marked correctly

**Severity:** High

Any variable or output that carries a secret must be marked `sensitive = true`.

```terraform
# bad
variable "db_password" {
  type = string
}

# good
variable "db_password" {
  type      = string
  sensitive = true
}

output "connection_string" {
  value     = "...${var.db_password}..."
  sensitive = true
}
```

**Remediation:** Add `sensitive = true`. Note that this only suppresses display — the value is still in state, so encrypt your backend.

---

#### Secrets from a secrets manager, not tfvars

**Severity:** High

Secrets must not be passed via `.tfvars` files. Retrieve them from AWS Secrets Manager, SSM Parameter Store (SecureString), or Vault at apply time.

```terraform
# bad — password in a var file
db_password = "supersecret"

# good — retrieved at apply time
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```

**Remediation:** Move secrets into a secrets manager. Update CI to inject credentials via environment variables rather than var files.

---

### IAM and Permissions

#### No wildcard actions in IAM policies

**Severity:** Critical

`Action: "*"` grants every IAM action in every service. It must not appear in any customer-managed policy.

```terraform
# bad
data "aws_iam_policy_document" "too_broad" {
  statement {
    actions   = ["*"]
    resources = ["*"]
  }
}

# good
data "aws_iam_policy_document" "scoped" {
  statement {
    actions = [
      "s3:GetObject",
      "s3:PutObject",
    ]
    resources = [aws_s3_bucket.app.arn, "${aws_s3_bucket.app.arn}/*"]
  }
}
```

**Remediation:** Replace `*` with the minimum set of actions the principal actually needs. Use IAM Access Analyzer to generate least-privilege policies from real usage.

---

#### No wildcard resources in IAM policies

**Severity:** High

`Resource: "*"` applies an action to every resource in the account. Scope resources to specific ARNs or ARN patterns.

```terraform
# bad
statement {
  actions   = ["s3:GetObject"]
  resources = ["*"]
}

# good
statement {
  actions   = ["s3:GetObject"]
  resources = ["${aws_s3_bucket.app.arn}/*"]
}
```

**Remediation:** Replace `*` with the ARN of the intended resource. Use `${aws_s3_bucket.example.arn}/*` for bucket contents.

---

#### No inline IAM policies

**Severity:** Medium

Inline policies are invisible to IAM policy search and cannot be reused or attached to other principals. Use managed policies.

```terraform
# bad
resource "aws_iam_role_policy" "inline" {
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.app.json
}

# good
resource "aws_iam_policy" "app" {
  name   = "app-policy"
  policy = data.aws_iam_policy_document.app.json
}

resource "aws_iam_role_policy_attachment" "app" {
  role       = aws_iam_role.app.name
  policy_arn = aws_iam_policy.app.arn
}
```

**Remediation:** Convert inline policies to managed policies and attach them via `aws_iam_role_policy_attachment`.

---

#### Role trust policies scoped to specific principals

**Severity:** High

A trust policy with `Principal: "*"` allows any entity in the world to assume the role.

```terraform
# bad
statement {
  principals {
    type        = "*"
    identifiers = ["*"]
  }
  actions = ["sts:AssumeRole"]
}

# good
statement {
  principals {
    type        = "Service"
    identifiers = ["lambda.amazonaws.com"]
  }
  actions = ["sts:AssumeRole"]
}
```

**Remediation:** Scope the principal to the specific service, account, or role that needs to assume this role. Add `Condition` blocks to restrict further (e.g. `aws:SourceAccount`).

---

### Encryption

#### Encryption at rest on all storage resources

**Severity:** High

Every storage resource must have encryption at rest enabled. The following resources require explicit configuration:

| Resource | Required attribute |
|---|---|
| `aws_s3_bucket` | `aws_s3_bucket_server_side_encryption_configuration` |
| `aws_ebs_volume` | `encrypted = true` |
| `aws_db_instance` | `storage_encrypted = true` |
| `aws_rds_cluster` | `storage_encrypted = true` |
| `aws_dynamodb_table` | `server_side_encryption { enabled = true }` |
| `aws_sqs_queue` | `sqs_managed_sse_enabled = true` or `kms_master_key_id` |
| `aws_cloudwatch_log_group` | `kms_key_id` |
| `aws_elasticache_replication_group` | `at_rest_encryption_enabled = true` |

```terraform
# bad
resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = "db.t3.medium"
}

# good
resource "aws_db_instance" "main" {
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
}
```

**Remediation:** Add the missing encryption attribute. Prefer customer-managed KMS keys over AWS-managed keys for environments with compliance requirements.

---

#### Encryption in transit

**Severity:** High

Services should enforce TLS and reject unencrypted connections.

```terraform
# bad — allows unencrypted connections
resource "aws_elasticache_replication_group" "this" {
  transit_encryption_enabled = false
}

# good
resource "aws_elasticache_replication_group" "this" {
  transit_encryption_enabled = true
  auth_token                 = var.auth_token
}
```

For load balancers, redirect HTTP to HTTPS rather than serving on port 80:

```terraform
resource "aws_lb_listener" "http" {
  port     = 80
  protocol = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

#### Customer-managed KMS keys for sensitive workloads

**Severity:** Medium

AWS-managed keys cannot have key policies customised, cannot be rotated on demand, and do not provide the audit trail that CMKs do.

```terraform
resource "aws_kms_key" "main" {
  description             = "CMK for ${var.project} encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  tags                    = var.common_tags
}

resource "aws_kms_alias" "main" {
  name          = "alias/${var.project}-main"
  target_key_id = aws_kms_key.main.key_id
}
```

**Remediation:** Create a KMS CMK with `enable_key_rotation = true` and reference its ARN in storage resource encryption attributes.

---

### Network Security

#### No unrestricted ingress on sensitive ports

**Severity:** Critical

Security groups must not allow `0.0.0.0/0` or `::/0` ingress on administrative or database ports.

Sensitive ports: SSH (22), RDP (3389), PostgreSQL (5432), MySQL (3306), MSSQL (1433), Redis (6379), MongoDB (27017), Elasticsearch (9200, 9300).

```terraform
# bad
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

# good — restrict to VPN CIDR or use SSM Session Manager instead
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = [var.vpn_cidr]
}
```

**Remediation:** Replace `0.0.0.0/0` with the specific CIDR or security group ID of the source. For EC2 admin access, prefer AWS Systems Manager Session Manager over SSH entirely.

---

#### No unrestricted ingress on any port

**Severity:** High

Even for non-sensitive ports, `0.0.0.0/0` ingress should be avoided unless the resource is a public-facing load balancer.

```terraform
# acceptable only for a public ALB on 80/443
ingress {
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

For anything behind a load balancer, restrict ingress to the ALB security group:

```terraform
ingress {
  from_port       = 8080
  to_port         = 8080
  protocol        = "tcp"
  security_groups = [aws_security_group.alb.id]
}
```

---

#### VPC endpoints for AWS service access

**Severity:** Medium

Traffic to AWS services (S3, DynamoDB, SSM, Secrets Manager, ECR, etc.) should route via VPC endpoints rather than over the public internet.

```terraform
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id
  tags = var.common_tags
}
```

**Remediation:** Create gateway endpoints for S3 and DynamoDB (free). Create interface endpoints for other services. Update route tables and security groups as needed.

---

#### VPC flow logs enabled

**Severity:** Medium

Flow logs record accepted and rejected traffic at the VPC, subnet, or ENI level. Required for incident investigation and many compliance frameworks.

```terraform
resource "aws_flow_log" "vpc" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
}
```

---

#### Subnets do not auto-assign public IPs

**Severity:** Medium

```terraform
# bad
resource "aws_subnet" "app" {
  map_public_ip_on_launch = true
}

# good — only public-facing subnets should assign public IPs
resource "aws_subnet" "app" {
  map_public_ip_on_launch = false
}
```

---

### Logging and Auditing

#### CloudTrail enabled with log file validation

**Severity:** High

CloudTrail provides an audit trail of all API calls. Log file validation detects tampering.

```terraform
resource "aws_cloudtrail" "main" {
  name                          = "${var.project}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  kms_key_id                    = aws_kms_key.cloudtrail.arn
  tags                          = var.common_tags
}
```

**Remediation:** Create a CloudTrail that covers all regions, logs to an encrypted S3 bucket, and has log file validation enabled.

---

#### S3 access logging enabled

**Severity:** Medium

S3 server access logs record every request made to a bucket. Required for access auditing and incident investigation.

```terraform
resource "aws_s3_bucket_logging" "app" {
  bucket        = aws_s3_bucket.app.id
  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "s3-access/${aws_s3_bucket.app.id}/"
}
```

---

#### Log group retention periods set

**Severity:** Low

CloudWatch log groups with no retention period retain logs indefinitely, which increases cost and may violate data retention policies.

```terraform
# bad — no retention
resource "aws_cloudwatch_log_group" "app" {
  name = "/aws/lambda/${var.function_name}"
}

# good
resource "aws_cloudwatch_log_group" "app" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = 90
  kms_key_id        = aws_kms_key.logs.arn
  tags              = var.common_tags
}
```

---

### S3 Specific

#### Public access block configured on all buckets

**Severity:** Critical

Every S3 bucket must have all four public access block settings enabled unless it is intentionally a public static website.

```terraform
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**Remediation:** Add `aws_s3_bucket_public_access_block` for every `aws_s3_bucket`. For public websites, use CloudFront with OAC instead of direct bucket public access.

---

#### S3 bucket versioning enabled for state and critical buckets

**Severity:** High

Versioning enables recovery from accidental deletion or overwrite. Required for state buckets; recommended for any bucket storing application data.

```terraform
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

---

#### S3 bucket policy denies non-TLS requests

**Severity:** High

Without an explicit deny, objects can be accessed over HTTP.

```terraform
data "aws_iam_policy_document" "force_tls" {
  statement {
    sid     = "DenyNonTLS"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.this.arn,
      "${aws_s3_bucket.this.arn}/*",
    ]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_s3_bucket_policy" "force_tls" {
  bucket = aws_s3_bucket.this.id
  policy = data.aws_iam_policy_document.force_tls.json
}
```

---

### Stateful Resource Protection

#### `prevent_destroy` on critical resources

**Severity:** High

Databases, S3 buckets holding persistent data, KMS keys, and state buckets must not be destroyable by a routine `terraform destroy`.

```terraform
resource "aws_db_instance" "main" {
  lifecycle {
    prevent_destroy = true
  }
}
```

**Remediation:** Add `lifecycle { prevent_destroy = true }` to every stateful resource. Enable deletion protection at the AWS API level as well (e.g. `deletion_protection = true` on RDS, `force_destroy = false` on S3).

---

#### Deletion protection at the resource level

**Severity:** High

Some resources support API-level deletion protection independent of Terraform's `prevent_destroy`.

```terraform
# RDS
resource "aws_db_instance" "main" {
  deletion_protection = true
}

# ALB
resource "aws_lb" "main" {
  enable_deletion_protection = true
}
```

---

### Supply Chain

#### Provider versions pinned and lock file committed

**Severity:** High

Floating provider versions (`>= 5.0` without an upper bound, or no version at all) can introduce breaking changes or malicious updates between runs.

```terraform
# bad — no upper bound
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# good
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

`.terraform.lock.hcl` must be present in the repository and committed. Absence of the lock file means provider checksums are not verified between runs.

---

#### Module sources pinned to a specific version or SHA

**Severity:** High

Modules referenced without a version or with a floating branch reference can silently pull in upstream changes.

```terraform
# bad — floating branch
module "vpc" {
  source = "git::https://github.com/example/terraform-aws-vpc.git?ref=main"
}

# good — pinned tag
module "vpc" {
  source  = "jameswoolfenden/vpc/aws"
  version = "1.2.3"
}
```

**Remediation:** Pin registry modules with `version`. Pin git source modules with a tag or full commit SHA in the `ref` parameter.

---

#### Third-party modules from trusted sources only

**Severity:** Medium

Before using a community module, verify:

- The module is published by a verified publisher or a trusted organisation
- The source repository is active and maintained
- The module does not request excessive permissions
- The module version is pinned (see above)

Prefer official HashiCorp, AWS, or well-known community modules (e.g. [terraform-aws-modules](https://github.com/terraform-aws-modules)) over unknown publishers.

---

### State Security

#### Remote state backend in use

**Severity:** Critical

Local state must not be used in shared or production environments (see the [State](#state) section).

**Check:** `terraform.tf` or `backend.tf` must contain a `backend` block other than `local`.

```terraform
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "tf-state-lock"
  }
}
```

---

#### State backend encrypted

**Severity:** Critical

The S3 backend's `encrypt = true` flag must be set. The state bucket itself must also have server-side encryption configured (see [Encryption at rest](#encryption-at-rest-on-all-storage-resources)).

---

#### State backend access restricted

**Severity:** High

The S3 bucket used for state should:

- Block all public access
- Enforce TLS via bucket policy
- Restrict `s3:GetObject` and `s3:PutObject` to the specific IAM roles that run Terraform
- Enable versioning for rollback capability

The DynamoDB table used for locking should be encrypted and access-restricted similarly.

---

## GCP Security Review

The same severity scale applies: **Critical** | **High** | **Medium** | **Low**

GCP Terraform resources use the `google` and `google-beta` providers. Checks below reference `google_*` resource types.

---

### Secrets and Credentials

#### No hardcoded service account keys

**Severity:** Critical

Service account key JSON must never appear in Terraform code or var files. Keys are long-lived credentials that bypass IAM conditions.

```terraform
# bad
resource "google_service_account_key" "app" {
  service_account_id = google_service_account.app.name
}
# then used as a literal in another resource — never do this

# good — use Workload Identity instead (see below)
```

**Remediation:** Delete existing keys. Use Workload Identity Federation for GKE workloads, or attach a service account to the compute resource directly. For external systems, use Workload Identity Federation with short-lived tokens.

---

#### Workload Identity instead of service account keys for GKE

**Severity:** High

Pods should never mount a service account key file. Workload Identity binds a Kubernetes service account to a GCP service account using short-lived tokens.

```terraform
# good
resource "google_service_account" "app" {
  account_id   = "app-sa"
  display_name = "App Service Account"
}

resource "google_service_account_iam_member" "workload_identity" {
  service_account_id = google_service_account.app.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project}.svc.id.goog[${var.namespace}/${var.ksa_name}]"
}

resource "google_container_cluster" "main" {
  workload_identity_config {
    workload_pool = "${var.project}.svc.id.goog"
  }
}
```

---

#### Secrets in Secret Manager, not variable files

**Severity:** High

```terraform
# bad — password in tfvars
db_password = "supersecret"

# good
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password"
  replication {
    auto {}
  }
}

data "google_secret_manager_secret_version" "db_password" {
  secret = google_secret_manager_secret.db_password.id
}
```

---

### IAM and Permissions

#### No primitive roles at project level

**Severity:** Critical

The primitive roles `roles/owner`, `roles/editor`, and `roles/viewer` are coarse-grained and grant access far beyond any single service. Never assign them at the project level.

```terraform
# bad
resource "google_project_iam_member" "too_broad" {
  project = var.project
  role    = "roles/editor"
  member  = "serviceAccount:${google_service_account.app.email}"
}

# good — use a predefined or custom role scoped to what the SA needs
resource "google_project_iam_member" "scoped" {
  project = var.project
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

**Remediation:** Replace primitive roles with predefined roles. Where no predefined role fits, create a `google_project_iam_custom_role` with only the required permissions.

---

#### No `allUsers` or `allAuthenticatedUsers` in IAM bindings

**Severity:** Critical

These members make resources public. They must not appear in any IAM binding except intentionally public GCS buckets (which should instead use `public_access_prevention`).

```terraform
# bad
resource "google_storage_bucket_iam_member" "public" {
  bucket = google_storage_bucket.app.name
  role   = "roles/storage.objectViewer"
  member = "allUsers"
}

# good — grant to specific service account or group
resource "google_storage_bucket_iam_member" "app" {
  bucket = google_storage_bucket.app.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${google_service_account.app.email}"
}
```

---

#### Prefer `google_*_iam_member` over `google_*_iam_binding`

**Severity:** Medium

`iam_binding` is authoritative for a role — it removes any members not listed in Terraform, including manually added ones. This can silently revoke access. Use `iam_member` for additive grants.

```terraform
# bad — authoritative, wipes other members of this role
resource "google_project_iam_binding" "app" {
  project = var.project
  role    = "roles/storage.objectViewer"
  members = ["serviceAccount:${google_service_account.app.email}"]
}

# good — additive
resource "google_project_iam_member" "app" {
  project = var.project
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

---

#### Service accounts scoped per workload

**Severity:** Medium

Do not reuse a single service account across multiple workloads. Each service should have its own SA with only the permissions it needs.

```terraform
resource "google_service_account" "cloud_run_api" {
  account_id   = "cloud-run-api"
  display_name = "Cloud Run API Service Account"
  project      = var.project
}
```

---

### Encryption

#### CMEK on all storage resources

**Severity:** High

GCP services use Google-managed encryption by default. For regulated workloads, use customer-managed encryption keys (CMEK) via Cloud KMS.

| Resource | CMEK attribute |
|---|---|
| `google_storage_bucket` | `encryption { default_kms_key_name }` |
| `google_bigquery_dataset` | `default_encryption_configuration { kms_key_name }` |
| `google_sql_database_instance` | `encryption_key_name` |
| `google_pubsub_topic` | `kms_key_name` |
| `google_container_cluster` | `database_encryption { key_name }` |
| `google_artifact_registry_repository` | `kms_key_name` |

```terraform
resource "google_kms_key_ring" "main" {
  name     = "${var.project}-keyring"
  location = var.region
}

resource "google_kms_crypto_key" "storage" {
  name            = "storage-key"
  key_ring        = google_kms_key_ring.main.id
  rotation_period = "7776000s" # 90 days
}

resource "google_storage_bucket" "app" {
  name     = "${var.project}-app"
  location = var.region

  encryption {
    default_kms_key_name = google_kms_crypto_key.storage.id
  }
}
```

---

#### KMS key rotation enabled

**Severity:** High

```terraform
resource "google_kms_crypto_key" "main" {
  name            = "main-key"
  key_ring        = google_kms_key_ring.main.id
  rotation_period = "7776000s" # 90 days — adjust to compliance requirement
}
```

**Remediation:** Set `rotation_period` on every `google_kms_crypto_key`. NIST recommends rotation at least annually; most compliance frameworks require 90 days or less.

---

### Network Security

#### No `0.0.0.0/0` source in firewall rules on sensitive ports

**Severity:** Critical

```terraform
# bad
resource "google_compute_firewall" "ssh_open" {
  name    = "allow-ssh"
  network = google_compute_network.main.id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
}

# good — use Identity-Aware Proxy for SSH/RDP; no firewall rule needed
# If a rule is required, restrict to the IAP CIDR:
resource "google_compute_firewall" "ssh_iap" {
  name    = "allow-ssh-iap"
  network = google_compute_network.main.id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["35.235.240.0/20"] # IAP CIDR
  target_tags   = ["iap-ssh"]
}
```

**Remediation:** Use Identity-Aware Proxy for admin access to VMs. Remove all firewall rules permitting `0.0.0.0/0` on ports 22, 3389, and database ports.

---

#### Compute instances have no external IPs

**Severity:** High

Instances in private subnets should not have external IPs. Use Cloud NAT for outbound internet access.

```terraform
# bad
resource "google_compute_instance" "app" {
  network_interface {
    network = google_compute_network.main.id
    access_config {} # this assigns an ephemeral external IP
  }
}

# good
resource "google_compute_instance" "app" {
  network_interface {
    network    = google_compute_network.main.id
    subnetwork = google_compute_subnetwork.private.id
    # no access_config block = no external IP
  }
}
```

---

#### Private Google Access enabled on subnets

**Severity:** Medium

Private Google Access allows instances without external IPs to reach Google APIs and services over Google's internal network.

```terraform
resource "google_compute_subnetwork" "private" {
  name                     = "private"
  ip_cidr_range            = "10.0.0.0/24"
  region                   = var.region
  network                  = google_compute_network.main.id
  private_ip_google_access = true
}
```

---

#### VPC Service Controls for sensitive APIs

**Severity:** Medium

VPC Service Controls create a security perimeter around GCP services, preventing data exfiltration even if a service account is compromised.

```terraform
resource "google_access_context_manager_service_perimeter" "main" {
  parent = "accessPolicies/${var.access_policy_id}"
  name   = "accessPolicies/${var.access_policy_id}/servicePerimeters/main"
  title  = "main"

  status {
    restricted_services = [
      "bigquery.googleapis.com",
      "storage.googleapis.com",
    ]
    resources = ["projects/${var.project_number}"]
  }
}
```

---

### Logging and Auditing

#### Data Access audit logs enabled

**Severity:** High

GCP audit logs have three categories: Admin Activity (always on), Data Access (off by default), and System Events. Data Access logs must be explicitly enabled for sensitive services.

```terraform
resource "google_project_iam_audit_config" "main" {
  project = var.project
  service = "allServices"

  audit_log_config {
    log_type = "ADMIN_READ"
  }

  audit_log_config {
    log_type = "DATA_READ"
  }

  audit_log_config {
    log_type = "DATA_WRITE"
  }
}
```

**Remediation:** Enable Data Read and Data Write audit logs for at minimum: `storage.googleapis.com`, `bigquery.googleapis.com`, `secretmanager.googleapis.com`, `cloudkms.googleapis.com`.

---

#### Log sink with long-term retention

**Severity:** Medium

Cloud Logging default retention is 30–400 days depending on log type. Export to GCS or BigQuery for longer retention and compliance.

```terraform
resource "google_logging_project_sink" "audit" {
  name        = "audit-sink"
  destination = "storage.googleapis.com/${google_storage_bucket.audit_logs.name}"
  filter      = "logName:(cloudaudit.googleapis.com)"

  unique_writer_identity = true
}

resource "google_storage_bucket_iam_member" "log_writer" {
  bucket = google_storage_bucket.audit_logs.name
  role   = "roles/storage.objectCreator"
  member = google_logging_project_sink.audit.writer_identity
}
```

---

### GCS Specific

#### Uniform bucket-level access enforced

**Severity:** High

Uniform bucket-level access disables object-level ACLs and ensures all access is controlled through IAM only. Without it, objects can be made public via legacy ACLs regardless of IAM policy.

```terraform
resource "google_storage_bucket" "app" {
  name                        = "${var.project}-app"
  location                    = var.region
  uniform_bucket_level_access = true
}
```

---

#### Public access prevention enforced

**Severity:** Critical

```terraform
resource "google_storage_bucket" "app" {
  name     = "${var.project}-app"
  location = var.region

  public_access_prevention    = "enforced"
  uniform_bucket_level_access = true
}
```

**Remediation:** Set `public_access_prevention = "enforced"` on every bucket that does not serve public content. For public static sites, use Cloud CDN with a load balancer rather than direct public bucket access.

---

#### Bucket versioning for persistent data

**Severity:** Medium

```terraform
resource "google_storage_bucket" "data" {
  name     = "${var.project}-data"
  location = var.region

  versioning {
    enabled = true
  }

  lifecycle_rule {
    action { type = "Delete" }
    condition {
      num_newer_versions = 10
      with_state         = "ARCHIVED"
    }
  }
}
```

---

#### Bucket retention policy for audit logs

**Severity:** Medium

Retention policies prevent log objects from being deleted before the retention period expires, even by bucket owners.

```terraform
resource "google_storage_bucket" "audit_logs" {
  name     = "${var.project}-audit-logs"
  location = var.region

  retention_policy {
    retention_period = 31536000 # 1 year in seconds
    is_locked        = true     # prevents policy from being weakened
  }

  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
}
```

---

### Cloud SQL

#### No public IP on Cloud SQL instances

**Severity:** Critical

Cloud SQL instances must not have a public IPv4 address. Use Cloud SQL Auth Proxy or Private Service Connect for access.

```terraform
# bad
resource "google_sql_database_instance" "main" {
  settings {
    ip_configuration {
      ipv4_enabled = true
    }
  }
}

# good
resource "google_sql_database_instance" "main" {
  settings {
    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }
  }
}
```

---

#### SSL required on Cloud SQL connections

**Severity:** High

```terraform
resource "google_sql_database_instance" "main" {
  settings {
    ip_configuration {
      require_ssl = true
    }
  }
}
```

---

#### Deletion protection and backups enabled

**Severity:** High

```terraform
resource "google_sql_database_instance" "main" {
  deletion_protection = true

  settings {
    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
    }
  }
}
```

---

### GKE

#### Private cluster with no public endpoint

**Severity:** High

```terraform
resource "google_container_cluster" "main" {
  name     = "${var.project}-cluster"
  location = var.region

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = true
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = var.vpn_cidr
      display_name = "VPN"
    }
  }
}
```

---

#### Shielded nodes enabled

**Severity:** Medium

Shielded GKE nodes use Secure Boot, vTPM, and Integrity Monitoring to prevent root-level tampering.

```terraform
resource "google_container_node_pool" "main" {
  node_config {
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }
}
```

---

#### Network policy enabled

**Severity:** Medium

Without a network policy, any pod in the cluster can communicate with any other pod. Enable Calico or Dataplane V2.

```terraform
resource "google_container_cluster" "main" {
  network_policy {
    enabled  = true
    provider = "CALICO"
  }

  # Dataplane V2 (preferred — includes network policy enforcement)
  datapath_provider = "ADVANCED_DATAPATH"
}
```

---

#### Binary Authorization enabled

**Severity:** Medium

Binary Authorization ensures only images that have been attested (e.g. passed a CI security scan) can be deployed to GKE.

```terraform
resource "google_container_cluster" "main" {
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }
}

resource "google_binary_authorization_policy" "main" {
  default_admission_rule {
    evaluation_mode  = "REQUIRE_ATTESTATION"
    enforcement_mode = "ENFORCED_BLOCK_AND_AUDIT_LOG"
    require_attestations_by = [google_binary_authorization_attestor.ci.name]
  }
}
```

---

## Configuration

Convention over configuration is preferred.
Use a data source over adding a configuration value where possible.
Set default values for module variables.
Make resources optional using `count = var.enabled ? 1 : 0` or `for_each` over a conditionally empty map/set.

---

## Unit Testing

Use the native [Terraform test framework](https://developer.hashicorp.com/terraform/language/tests) (available from Terraform 1.6+). Tests live in `.tftest.hcl` files alongside the module.

```hcl
# tests/defaults.tftest.hcl
run "creates_bucket" {
  command = apply

  variables {
    bucket_name = "test-bucket-${run.creates_bucket.id}"
    environment = "test"
  }

  assert {
    condition     = aws_s3_bucket.this.bucket == "test-bucket-${run.creates_bucket.id}"
    error_message = "Bucket name did not match expected value."
  }
}
```

Run with:

```bash
terraform test
```

For existing modules not yet using the test framework:

- Include a runnable example in `examples/examplea`
- Run init + validate + plan in CI for every PR
- Apply and destroy as part of a scheduled or post-merge pipeline
- Tag the successful build

Use [Checkov](https://checkov.io) for static security analysis. Consider [Open Policy Agent](https://www.openpolicyagent.org/) for custom policy enforcement.

---

## Tagging

Implement a tagging scheme from the start, and use a map type for extensibility.

In **variables.tf**:

```terraform
variable "common_tags" {
  type        = map(string)
  description = "Tags applied to every taggable resource"
}
```

In **locals.tf**, merge caller-supplied tags with computed tags so the module always adds its own required tags:

```terraform
locals {
  tags = merge(var.common_tags, {
    managed_by = "terraform"
    module     = "terraform-aws-example"
  })
}
```

In **example.auto.tfvars**:

```terraform
common_tags = {
  owner      = "platform-team"
  cost_code  = "4873"
  project    = "sap-proxy"
  team       = "Wilbur"
}
```

Apply `local.tags` on every taggable resource:

```terraform
resource "aws_s3_bucket" "state" {
  bucket = local.bucket_name
  tags   = local.tags
}
```

Names you can't update; tags you can. The longer you make resource names the more bugs you will find. Use the [Cloud Posse null-label module](https://github.com/cloudposse/terraform-null-label) if you must enforce a naming convention.

---

## Workspaces

### Terraform CLI workspaces

Avoid using Terraform CLI workspaces for environment separation — they were not designed for it. They share backend configuration and require workspace-conditional logic that makes code hard to read:

```terraform
# avoid this pattern
bucket = terraform.workspace == "preprod" ? var.bucket_preprod : var.bucket_prod
```

The correct alternative is separate state files per environment (see [Environments](#environments)).

CLI workspaces are appropriate for short-lived feature branches where a developer needs an isolated state for experimentation, not for long-lived prod/staging/dev separation.

### Terraform Cloud workspaces

Terraform Cloud workspaces are a different concept — each workspace is a separate environment with its own state, variables, and run history. These are the right tool for environment separation in Terraform Cloud.

See the [recommended practices guide](https://www.terraform.io/docs/cloud/guides/recommended-practices/part1.html).

---

## Recommended Tools

[terraform-docs](https://github.com/terraform-docs/terraform-docs) — generates README tables for inputs and outputs automatically.

[Pre-commit](https://pre-commit.com/) — runs linting, formatting, and security checks on every commit. Every Terraform repo should have one.

[Checkov](https://checkov.io) — static analysis for Terraform security misconfigurations. Make it part of CI.

[Trivy](https://github.com/aquasecurity/trivy) — broader security scanner that covers Terraform IaC alongside container images and packages.

[tflint](https://github.com/terraform-linters/tflint) — linter for Terraform that catches provider-specific issues (e.g. invalid instance types) that `terraform validate` misses.

[LocalStack](https://localstack.cloud/) — mock AWS environment for local development. Works with Terraform via the `awslocal` wrapper.

[SAML2AWS](https://github.com/Versent/saml2aws) — generates temporary AWS credentials from a federated identity provider. Essential in federated AD environments.

[GitHub Actions](https://github.com/features/actions) — built-in CI/CD for GitHub repositories. The natural first choice.

[GitHub CLI](https://cli.github.com/) — makes working with GitHub at the command line easy.

[VSCode](https://code.visualstudio.com/) with the [HashiCorp Terraform extension](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform) — syntax highlighting, auto-complete, and formatting.

[Beyond-Compare](https://www.scootersoftware.com/) or equivalent — cross-platform diff tool.

[mingrammer/diagrams](https://diagrams.mingrammer.com/) — visualise infrastructure designs as code.

![stateful architecture](https://diagrams.mingrammer.com/img/stateful_architecture_diagram.png)

---

## Caches

Set a [plugin cache](https://www.terraform.io/docs/commands/cli-config.html) in `~/.terraformrc`:

```ini
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

On a fast connection you may not notice repeated provider downloads, but on CI or a slow link the savings are significant. In CI, cache the plugin directory between runs using your CI system's cache action.

---

### Contributors

[![James Woolfenden][jameswoolfenden_avatar]][jameswoolfenden_homepage]<br/>[James Woolfenden][jameswoolfenden_homepage]

[jameswoolfenden_homepage]: https://github.com/jameswoolfenden
[jameswoolfenden_avatar]: https://github.com/jameswoolfenden.png?size=150
