---
description: |
  Scans a single Terraform repository for outdated module references and an
  outdated azurerm provider, upgrades and validates them, reviews the code
  against the shared Terraform coding standards, refreshes the documentation
  and opens a draft pull request in that repository.

on:
  workflow_dispatch:
    inputs:
      target_repo:
        description: "Target repository (owner/repo). Must be listed in config/targets.json."
        required: true
        type: string
      allow_major_upgrades:
        description: "Allow major version upgrades of modules and the azurerm provider."
        required: false
        type: boolean
        default: true

run-name: Terraform dependency update for ${{ github.event.inputs.target_repo }}

concurrency:
  group: gh-aw-${{ github.workflow }}-${{ github.event.inputs.target_repo }}
  job-discriminator: ${{ github.event.inputs.target_repo }}

engine: copilot

timeout-minutes: 30

permissions:
  contents: read
  issues: read
  pull-requests: read

imports:
  - shared/terraform-coding-standards.md

jobs:
  validate_target:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout target registry
        uses: actions/checkout@v7
        with:
          sparse-checkout: config/targets.json
          sparse-checkout-cone-mode: false
      - name: Ensure target repository is registered
        env:
          TARGET_REPO: ${{ github.event.inputs.target_repo }}
        run: |
          if ! jq -e --arg repo "$TARGET_REPO" '.repositories | index($repo)' config/targets.json > /dev/null; then
            echo "::error::'$TARGET_REPO' is not listed in config/targets.json"
            exit 1
          fi
  agent:
    needs: [validate_target]

checkout:
  repository: ${{ github.event.inputs.target_repo }}
  github-token: ${{ secrets.GH_AW_TARGET_REPO_READ_TOKEN }}
  current: true

network:
  allowed:
    - defaults
    - github
    - terraform

tools:
  github:
    github-token: ${{ secrets.GH_AW_TARGET_REPO_READ_TOKEN }}
    toolsets: [repos, pull_requests]
  web-fetch:
  edit:
  bash:
    - "terraform:*"
    - "git:*"
    - "find"
    - "ls"
    - "cat"
    - "grep"
    - "sed"
    - "jq"
    - "diff"

steps:
  - name: Setup Terraform
    uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
    with:
      terraform_wrapper: false
  - name: Expose Terraform to the agent sandbox
    run: sudo install -m 0755 "$(command -v terraform)" /usr/local/bin/terraform && terraform version

safe-outputs:
  github-token: ${{ secrets.GH_AW_CROSS_REPO_PAT }}
  create-pull-request:
    target-repo: ${{ github.event.inputs.target_repo }}
    title-prefix: "[terraform-deps] "
    labels: [dependencies, terraform, automated]
    draft: true
    max: 1
    if-no-changes: ignore
    excluded-files:
      - "**/.terraform/**"
      - "**/*.tfstate"
      - "**/*.tfstate.*"
      - "**/*.tfplan"
  noop:
---

# Terraform Dependency Update

You are a senior platform engineer maintaining Terraform modules for Microsoft Azure.
Your job is to keep **${{ github.event.inputs.target_repo }}** on current, validated
dependency versions and to deliver the change as a single, review-ready pull request.

The target repository is checked out in the current working directory.

- Target repository: `${{ github.event.inputs.target_repo }}`
- Major version upgrades allowed: `${{ github.event.inputs.allow_major_upgrades }}`

## Step 1 – Inventory

Find every Terraform file (`*.tf`, `*.tf.json`) in the repository, including `examples/`, `tests/`
and nested module folders, plus every Markdown file that contains HCL snippets (`README.md`,
`docs/**`, `examples/**/README.md`, `CHANGELOG.md`).

Record:

1. **Module references** – every `module` block `source` / `version`, and every module source
   string shown in documentation. Supported source formats:
   - Git: `git::https://github.com/<owner>/<repo>.git//<path>?ref=<tag>` and `github.com/<owner>/<repo>//<path>?ref=<tag>`
   - Terraform Registry: `<namespace>/<name>/<provider>` with a `version` argument
   - Local paths (`./`, `../`) – ignore, they are always current.
2. **Provider constraints** – every `required_providers` entry, with special attention to
   `hashicorp/azurerm`, plus `required_version` and any committed `.terraform.lock.hcl` files.

## Step 2 – Resolve the latest versions

- **Git modules on GitHub**: use the GitHub tools to list the releases of the source repository
  and pick the highest non-prerelease semantic version. If the repository has no releases, use
  the highest semantic version tag.
- **Registry modules**: fetch `https://registry.terraform.io/v1/modules/<namespace>/<name>/<provider>/versions`.
- **azurerm provider**: fetch `https://registry.terraform.io/v1/providers/hashicorp/azurerm/versions`
  and pick the highest non-prerelease version.

If major upgrades are **not** allowed, choose the highest version within the currently referenced
major version instead.

A reference is **outdated** when its pinned version (or the lower bound of its constraint) is lower
than the version selected above. If nothing is outdated and no `fix` action from the coding
standards applies, call `noop` with a short summary of the versions you checked and stop.

Before changing anything, use the GitHub tools to look for an open pull request in the target
repository whose title starts with `[terraform-deps]`. If it already proposes exactly the same
target versions, call `noop` referencing that pull request and stop.

## Step 3 – Apply the upgrade

1. **Module references** – update every outdated `?ref=` / `version` value in `.tf` files **and**
   in documentation snippets so code and docs stay in sync. For major upgrades, read the
   release notes / `CHANGELOG.md` of the upgraded module and adapt renamed, removed or newly
   required inputs and outputs.
2. **azurerm provider** – raise the constraint while keeping the existing style of the repository:
   - `~> X.Y` becomes `~> <latest major>.<latest minor>`
   - `>= X.Y.Z` / `>= X.Y.Z, < N.0` raises the lower bound to the latest version (and the upper bound to the next major when crossing a major)
   - Apply the same constraint in the root module and in every example / test configuration.
   - When crossing a major version, follow the official azurerm upgrade guide
     (`https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/<major>.0-upgrade-guide`)
     and migrate renamed or removed arguments and resources, for example `enable_*` → `*_enabled`
     argument renames and the `subscription_id` requirement in `provider "azurerm"` blocks of examples.
3. If a `.terraform.lock.hcl` is committed, refresh it with `terraform -chdir=<dir> init -upgrade -backend=false`.
   Do not add lock files to directories that did not have one.

## Step 4 – Validate

Run the following in the repository root **and** in every directory under `examples/` and `tests/`
that contains Terraform files:

```bash
terraform fmt -recursive
terraform -chdir=<dir> init -backend=false -upgrade -input=false
terraform -chdir=<dir> validate
```

- Fix every error introduced by the upgrade and rerun until validation succeeds.
- Never stage `.terraform/` folders, state files or plan files.
- If validation still fails after reasonable effort, do **not** open a pull request with broken
  code. Instead call `noop` explaining the blocking error, the attempted versions and the failing
  file(s).

## Step 5 – Coding standards

Apply the **Terraform Coding Standards** section to the changed repository. Perform the `fix`
actions in the same change set and collect all findings for the pull request body.

## Step 6 – Documentation

Update the documentation so it reflects the new state:

- `README.md` (root and examples): module usage snippets, `?ref=` / `version` values, provider and
  Terraform version requirements tables, inputs/outputs tables when they changed. If the README
  contains `<!-- BEGIN_TF_DOCS -->` / `<!-- END_TF_DOCS -->` markers, keep the markers and update
  the content between them in the same terraform-docs format.
- `CHANGELOG.md` (only if it exists): add the changes under an `Unreleased` section following the
  existing format.

## Step 7 – Pull request

Commit all changes on a branch named `terraform-deps/update-dependencies` and create **one**
pull request with:

- **Title**: `Update Terraform dependencies (azurerm <old> → <new>, <n> module reference(s))`
- **Body**:
  1. **Summary** – one paragraph describing what was updated and why.
  2. **Version changes** – table with `Dependency`, `Type` (module / provider), `From`, `To`, `Files`.
  3. **Breaking changes & migrations** – what was adapted for major upgrades (or "None").
  4. **Validation** – the commands you ran per directory and their result.
  5. **Coding standards review** – the table described in the coding standards section.
  6. **Documentation** – the documentation files that were updated.

Keep the change focused: do not refactor unrelated code, rename resources or change behaviour
beyond what the upgrade and the `fix` actions require.
