# A guide

[![Latest Release](https://img.shields.io/github/v/tag/jameswoolfenden/terraform-guidelines.svg)](https://github.com/jameswoolfenden/terraform-guidelines)
[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white)](https://github.com/pre-commit/pre-commit)

By [James Woolfenden](https://www.linkedin.com/in/jameswoolfenden/)

## A style guide for writing Terraform

This is a guide to writing Terraform, it follows the Hashicorp guide to creating modules for the [Terraform Registry](https://www.terraform.io/docs/registry/modules/publish.html) and their [standard structure](https://www.terraform.io/docs/modules/index.html#standard-module-structure).

There are many successful ways of writing your tf, this one is tried and field tested.

## Naming

Use Lower-case names for resources.
Use "\_" as a separator for resource names.
Names must be self explanatory containing several lower-case words if needed separated by "\_".

Use descriptive and non environment specific names to identify resources.

Avoid Tautologies:

``` terraform
resource "aws_iam_policy" "ec2_policy"{
  ...
}
```

## Hard-coding

Don't hard-code values in resources. Add variables and set defaults.

You can avoid limiting yourself with policies and resources by making resources optional or over-ridable.

```Terraform
resource "aws_iam_role" "codebuild" {
  name  = "codebuildrole-${var.name}"
  count = "${var.role == "" ? 1 : 0}"

  assume_role_policy = <<HERE
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "codebuild.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
HERE

  tags = var.common_tags
}
```

And avoid HEREDOCS like the one above, and use **data.aws_iam_policy_documents** objects, as practical.

## Templates

This is the Terraform code that is environment specific.  If your Templates are application specific the code should live with the code that requires it, create a folder in the root of the repository and call it **IAC**, similar to this for the repository aws-lexbot-handlers:

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

Inside the **iac** I breakdown the **templates** used:

```bash
total 0
drwxrwxrwx    1 jimw     jimw           512 May 28 11:22 codebuild
drwxrwxrwx    1 jimw     jimw           512 Apr  2 11:00 repository
```

This example  has an AWS CodeCommit repository (self describing) and an AWS Codebuild, that has multiple environments:

```bash
total 0
drwxrwxrwx    1 jimw     jimw           512 Apr 24 23:28 dev
drwxrwxrwx    1 jimw     jimw           512 Apr 24 23:29 prod
```

Inside each of these folder is an environmental specific template:

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

There's a lot of files here and some repetition - that violates DRY principles, but with IAC, favour on being explicit.
Each template is directly runnable, using the Terraform CLI with no wrapper script required.
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

### .github/workflow/main.yml

If your using Github you can add a basic CI workflow:

This is a sample workflow, for the minimum validation of a module **main.yml**:

```yml
---
name: Verify and Bump
on:
  push:
    branches:
      - master
jobs:
  build:

    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.7]

    steps:
      - uses: actions/checkout@v2
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v1
        with:
          python-version: ${{ matrix.python-version }}
      - name: "Terraform Init"
        uses: hashicorp/terraform-github-actions@master
        with:
          tf_actions_version: 0.12.28
          tf_actions_subcommand: "init"
          tf_actions_working_dir: "example/examplea"
      - name: "Terraform Validate"
        uses: hashicorp/terraform-github-actions@master
        with:
          tf_actions_version: 0.12.28
          tf_actions_subcommand: "validate"
          tf_actions_working_dir: "example/examplea"
      - name: Test with Checkov
        run: |
          pip install checkov
          checkov -d .
  version:
    name: versioning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - name: Bump version and push tag
        uses: anothrNick/github-tag-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DEFAULT_BUMP: patch
          WITH_V: "true"
    needs: build

```

### .terraformignore

If using Terraform cloud, the files not to upload to their cloud.

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

When you clone a file with hooks they are now installed.

And after that you need to add this file (or a similar one) to any new repository **pre-commit-config.yaml**, in the root:

```yaml
---
# yamllint disable rule:line-length
default_language_version:
  python: python3
repos:
  - repo: git://github.com/pre-commit/pre-commit-hooks
    rev: v3.2.0
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
  - repo: git://github.com/Lucas-C/pre-commit-hooks
    rev: v1.1.9
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
  - repo: git://github.com/jameswoolfenden/pre-commit-shell
    rev: 0.0.2
    hooks:
      - id: shell-lint
  - repo: git://github.com/igorshubovych/markdownlint-cli
    rev: v0.23.2
    hooks:
      - id: markdownlint
  - repo: git://github.com/adrienverge/yamllint
    rev: v1.24.2
    hooks:
      - id: yamllint
        name: yamllint
        description: This hook runs yamllint.
        entry: yamllint
        language: python
        types: [file, yaml]
  - repo: git://github.com/jameswoolfenden/pre-commit
    rev: v0.1.33
    hooks:
      - id: terraform-fmt
      - id: checkov-scan
        language_version: python3.7
      - id: tf2docs
        language_version: python3.7
```

Use hooks that don't clash and ensure they are cross platform compatible.

``` bash
pre-commit install
```

Hooks choices are a matter for each project, but if your are using Terraform or AWS the credentials and private key hooks or equivalent are required.

### main.tf

This is an expected file for Terraform modules. Remove it if this a template and add a **module.tf**.

### Makefile

A helper file. This is just to make like easier for you. Problematic if you are cross platform as make isn't very good/awful at that. If you use Windows update the PowerShell profile with equivalent helper functions instead.

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
   terraform destroy -force

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

A standard place to return values, either to the screen or to pass back from a module. Use descriptions.

### provider.aws.tf

Or Whatever you provider is or are.
Required. You are always going to be using these, included is this, the most basic provider for AWS.

### README.md

Where all the information goes.

### example.auto.tfvars

Files ending .auto.tfvars get picked by Terraform locally and in Terraform cloud.
This is the standard file for setting your variables in, and is automatically picked up by Terraform.

### variables.tf

For defining your variables and setting default values. Each variable should define its type and have an adequate description.
Also contains a map variable common_tags, which should be extended and used on every taggable object.  Use descriptions.

As of Terraform 0.13 Custom Input validation for variables has arrived.

Ok so? This means we finally has some control for Testing and Validation.

You can now automatically add tests so that your input values are set correctly, for your templates and modules, it a basic boundary check.

The minimal basics are explained here: https://www.terraform.io/docs/configuration/variables.html this is a very basic explanation.. sigh.

To see what really possible you need to combine these conditions with the functions that are already available in Terraform:

#### Check to see value is not null

```terraform
variable "config_file" {
  type        = string
  description = ""
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
  description = "The Log level value must be one of 'DEBUG', 'INFO','WARNING', 'ERROR','CRITICAL'."

  validation {
    condition     = can(regex("DEBUG|INFO|WARNING|ERROR|CRITICAL", var.log_level))
    error_message = "The Value must be one of 'DEBUG', 'INFO','WARNING', 'ERROR','CRITICAL'."
  }
}
```

or

```terraform
variable "tracing_mode" {
  type        = string
  description = "x-rays settings"
  default     = "Active"

  validation {
    condition     = contains(["PassThrough","Active"], var.tracing_mode)
    error_message = "Tracing mode can only be PassThrough or Active."
  }
}
```

#### Check to see variable is formatted correctly

```terraform
variable "runtime" {
  type        = string
  description = "The lambda runtime"
  default     = "python2.7"
  validation {
    condition     = length(var.runtime) > 5 && substr(var.runtime, 0, 6) == "python"
    error_message = "This uses python so its value need to start with \"python\"."
  }
}
```

### .dependsabot/config.yml

Sets the repository to be automatically dependency scanned in Github.

## Modules

You've written some TF and your about to duplicate its' functionality, it's time to abstract to a module. A module should be more than just one resource, it should add something.
Modules should be treated like applications services with a separate code repository for each module.

Each module should have a least one example included that demonstrates its usage. This example can be used as a test for that module, here its called **examplea**.

```bash
examples/examplea
```

This is an example for using AWS codecommit that conforms <https://github.com/JamesWoolfenden/terraform-aws-codecommit>

Use lowercase for all folder namesm, avoid spaces.

## Files

### Name your files after their contents

For a security group called "elastic", the resource is then aws_security_group.elastic, so the file is **aws_security_group.elastic.tf**. Be explicit.
It will save you time.

### Comments

Use Markdown for this, as many fmt and parsers break when you add comments into your TF with hashes and slash star comments. Some say it helps. Make sure to add descriptions to your variables and outputs.

### One resource per file

**Exception**: By all means group resources - where its really makes logical sense, security_group with rules, routes with route tables.

The style suggestion follows the standard practice in most development languages, it helps you navigate your code in your editor. Editing the same file in multiple locations is unhelpful.

## Be Specific

You have 2 choices with dependencies. Live on the bleeding edge, or fix your versions. I recommend being in control [That's fixing the version].

### Fix the version of Terraform you use

The whole team needs to use the same version of the tool until you decide as a team to update.
Create a file called **terraform.tf** in your template:

```terraform
terraform {
    required_version="~> 0.13.0"
}
```

You will need to do do a coordinated update of provider versions and tools versions at regular intervals e.g. ~3 months.

### Fix the version of the modules you consume

In your **module.tf** file, set the version of the module. If you author modules, make sure you tag successful module builds.
If your module comes from a registry, specify using the version property, if its only standard git source reference use a tag reference in your source statement.

If it's using modules from the registry like **modules.codebuild.tf**:

```terraform
module codebuild {
  source                 = "jameswoolfenden/codebuild/aws"
  version                = "0.1.41"
  root                   = var.root
  description            = var.description
}
```

### Fix the version of the providers you use

Using shiny things is great, what's not great is code that worked yesterday breaking because a plugin/provider changed. Specify the version in your **provider.tf** file.

```terraform
provider "aws" {
  region  = "eu-west-1"
  version = "3.00.0"
}
```

## State

By default, Terraform stores the state locally in a file named `terraform.tfstate`; a local state means that any changes applied to the infrastructure will only be aligned with the developer's machine.<br/>
For a complete beginner that could be a solution when trying out Terraform for the first time, but it certainly doesn't scale: if someone else wants to contribute to the code, things get misaligned very quickly, and what if the state file automagically disappears from the hard drive? You'll either destroy unexpectedly or lose control of your infrastructure.

Adding the state file to the SCM might seem to solve some of these problems, but it is never recommended: secret values and passwords might be stored in the state file, and those should never be commited to version control. That approach also relies on having commited and pulled the very latest version, which in practice is next to impossible.

The correct way is to use [remote states](https://www.terraform.io/docs/state/remote.html): in this case, the _state_ data is written to a remote data store and shared with all the team members and/or automation tools.<br/>
Another important feature is [state locking](https://www.terraform.io/docs/state/locking.html): it stops the state file being corrupted by multiple writes, and it also indicates to users if they are attempting to update the same code.<br/>
Furthermore, the concept of versioning comes handy: it allows to recover from file corruption and accidental upgrades, as well as giving traceable record of past configuration.

There are many options for implementing remote state, popular choices include:

- [S3 state bucket](https://www.terraform.io/docs/backends/types/s3.html) with Bucket Versioning and DynamoDB locking<br/>
  Quite common when using the AWS provider; a good template can be found [here](https://registry.terraform.io/modules/JamesWoolfenden/statebucket/aws/0.0.15)

- [GCS state bucket](https://www.terraform.io/docs/backends/types/gcs.html) with Object Versioning<br/>
  Used mainly when creating the infrastructure for the Google Cloud Platform

- [Azure state blob](https://www.terraform.io/docs/backends/types/azurerm.html)<br/>
  Stores the state using Azure Blob Storage, which supports locking and consistency checking<br/>

- [Terraform Cloud/Enterprise](https://www.terraform.io/docs/backends/types/remote.html#basic-configuration)<br/>
  A good option for provider-agnostic storage of the state; requires configuring the [access credentials](https://www.terraform.io/docs/commands/cli-config.html#credentials) (token) via a `terraform.rc` file

- ... even more [here](https://www.terraform.io/docs/backends/types/index.html)

A good choice for multi-provider code is Terraform Cloud: one key element to keep in mind is that the sensitive data part of the state will be stored on HashiCorp's servers.

For Terraform code that uses (primarily) one provider, a good option is to use the service-specific storage and locking method.

## Layout

Mandate the use of the standard pre-commits, this enforces the use of the command **Terraform fmt** on every Git commit. End of problem.

## Protecting Secrets

Protect your secrets by installing using the pre-commit file and the hooks from the standard set:

```hooks
- id: detect-aws-credentials
- id: detect-private-key
```

Other options include using git-secrets, Husky or using Talisman.  Use and mandate use of one, by all. Don't be that person.

## Configuration

Convention over configuration is preferred.
Use a data source over adding a configuration value.
Set default values for your modules variables.
Make resources optional with the count syntax.

## Unit Testing

As yet to find a really satisfactory test approach or tool for testing Terraform other than:

- Include a test implementation with your modules - from your examples root folder.
- Run it for every change.
- Tag the successful outcomes.
- Destroy created resources

Use Checkov, Consider using the Open Policy Agent.

## Tagging

Implement a tagging scheme from the start, and use a map type for extensibility.

In **variables.tf**:

```terraform
variable "common_tags" {
  type       = "map"
  description= "Implements the common_tags scheme"
}
```

And in your **example.auto.tfvars**

```terraform
  common_tags={
    name      = "sap-proxy-layer"
    owner     = "James Woolfenden"
    costcentre= "development"
  }
```

and then have the common_tags used in your resources file:

```terraform
resource "aws_codebuild_project" "project" {
  name          = "${replace(var.name,".","-")}"
  description   = var.description
  service_role  = "${var.role == "" ? element(concat(aws_iam_role.codebuild.*.arn, list("")), 0) : element(concat(data.aws_iam_role.existing.*.arn, list("")), 0) }"
  build_timeout = "var.build_timeout

  artifacts {
    type                = var.type
    location            = local.bucketname
    name                = var.name
    namespace_type      = var.namespace_type
    packaging           = var.packaging
    encryption_disabled = var.encryption_disabled
  }

  environment = var.environment
  source      = var.sourcecode
  tags        = var.common_tags
}
```

So, use Readable Key value pairs.
You don't have to name your like your still on premise.
So naming a security group

DEV_TEAM_WILBUR4873_APP_SG

is not necessarily helpful but

```HCL
tags={
TEAM="Wilbur"
Purpose="App"
CostCode="4873"}
```

Is better. Names you can't update, tags you can. The longer you make the resource names the more bugs you will find/make.
Ok I get it some resources don't have tag attributes or you have some "Security" policy or other that mean you must have a naming regime.

If so I'd either use or copy the naming module from the Cloud Posse
[https://github.com/cloudposse/terraform-null-label](https://github.com/cloudposse/terraform-null-label).

## Workspaces

Confusingly this means 2 things in the Terraform world.

### Old workspaces

When should I use these workspaces, here is the hashicorp guidance:

<https://www.terraform.io/docs/state/workspaces.html#when-to-use-multiple-workspaces>

Clearly a reduced use case of just using for branch based changes, but i think even that case is weak:
Workspace interpolation pretty much makes me want to say no to using. Workspaces aren't for separate environments and you have to code you vars and values special for using them.

How not to use this:

```cli
bucket = "${terraform.workspace == "preprod" ? var.bucket_demo_preprod : var.bucket_demo}"
```

Not simple.

<https://medium.com/@milescollier/handling-environmental-variables-in-terraform-workspaces-27d0278423df>

I'm sure you could make this work.

### New workspaces

These workspaces are concerned with Terraform Cloud:
<https://www.terraform.io/docs/cloud/guides/recommended-practices/part1.html>

Obvious.

## Recommended Tools

[Terraform-docs](https://github.com/segmentio/terraform-docs)

Run to help make your readmes.

The [Pre-commit](https://pre-commit.com/) framework

So many different uses from linting to security, every git repo should have one.

[Beyond-Compare](https://www.scootersoftware.com/) or equivalent

A preference, for a comparison tool. This one is cross platform compatible.

The Cli

Be it AWS, or whatever provider your using.

[VSCode](https://code.visualstudio.com/) and Extensions

Free and really quite good editor, with awesome extensions. Use the Extensions sync extension to maintain your [environment](https://gist.github.com/JamesWoolfenden/1a1ce363e6e6e5d2bcf321ca12ec3de2).

[SAML2AWS](https://github.com/Versent/saml2aws)

Generates temporary AWS credentials for AWS cmdline. Essential for running in Federated AD environment.

[Checkov](https://checkov.io)

Checks your Terraform for security flaws. This should be part of basic toolset.

[Travis](https://travis-ci.com/) - or free for [open source projects](https://travis-ci.org/getting_started).

[Circle](https://circleci.com/) a Leading CICD SAS tool that integrates with your codebase.

There are many other good SAS CI/CD tools including Circle, GitLab and a few shockers.

[AzureDevOps](https://azure.microsoft.com/en-gb/services/devops/)

It maybe born/spawn of TFS but this tool has improved so much (I KNOW) to make it a viable option even for non Azure projects.

[Hub](https://github.com/github/hub)

Makes working with Github at the cli easy.

[LocalStack](https://localstack.cloud/)

This tool allows you to mock your AWS set-up, works with Terraform and the awscli - and use the awslocal wrapper.

[mingrammer/diagrams](https://diagrams.mingrammer.com/)

Use this tool to visualise your designs as code.

![stateful architecture](https://diagrams.mingrammer.com/img/stateful_architecture_diagram.png)

See the repo <https://github.com/mingrammer/diagrams>.

## Caches

Set a [plugin cache](https://www.terraform.io/docs/commands/cli-config.html). On a fat pipe you might not notice how quickly they download, but do set-up your plugin-cache. It will save you time and stress.

### Contributors

[![James Woolfenden][jameswoolfenden_avatar]][jameswoolfenden_homepage]<br/>[James Woolfenden][jameswoolfenden_homepage]

[jameswoolfenden_homepage]: https://github.com/jameswoolfenden
[jameswoolfenden_avatar]: https://github.com/jameswoolfenden.png?size=150
