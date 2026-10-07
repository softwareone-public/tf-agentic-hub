---
description: |
  Scans a single Terraform repository for outdated module references, upgrades
  and validates them, reviews the code
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
        description: "Allow major version upgrades of Terraform modules."
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
  copilot-requests: write

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
  github-app:
    client-id: ${{ vars.APP_CLIENT_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}
    owner: softwareone-public
    repositories: ["${{ github.event.inputs.target_repo }}"]
  safe-outputs-github-app:
    client-id: ${{ vars.APP_CLIENT_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}
    owner: softwareone-public
    repositories: ["${{ github.event.inputs.target_repo }}"]
  current: true

network:
  allowed:
    - defaults
    - github
    - terraform

tools:
  github:
    github-app:
      client-id: ${{ vars.APP_CLIENT_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}
      owner: softwareone-public
      repositories: ["${{ github.event.inputs.target_repo }}"]
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
  github-app:
    client-id: ${{ vars.APP_CLIENT_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}
    owner: softwareone-public
    repositories: ["${{ github.event.inputs.target_repo }}"]
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
2. **Terraform configuration** – record `required_version` and committed `.terraform.lock.hcl`
  files for validation context. Review provider requirements only as part of the shared coding
  standards; do not resolve their latest versions.

## Step 2 – Resolve the latest versions

- **Git modules on GitHub**: use the GitHub tools to list the releases of the source repository
  and pick the highest non-prerelease semantic version. If the repository has no releases, use
  the highest semantic version tag.
- **Registry modules**: fetch `https://registry.terraform.io/v1/modules/<namespace>/<name>/<provider>/versions`.

If major upgrades are **not** allowed, choose the highest version within the currently referenced
major version for each module instead.

A module reference is **outdated** when its pinned version is lower than the version selected above.
If no module is outdated and no `fix` action from the coding standards applies, call `noop` with a
short summary of the module versions you checked and stop.

Before changing anything, use the GitHub tools to look for an open pull request in the target
repository whose title starts with `[terraform-deps]`. If it already proposes exactly the same
module versions, call `noop` referencing that pull request and stop.

## Step 3 – Apply the upgrade

1. **Module references** – update every outdated `?ref=` / `version` value in `.tf` files **and**
   in documentation snippets so code and docs stay in sync. For major upgrades, read the
   release notes / `CHANGELOG.md` of the upgraded module and adapt renamed, removed or newly
   required inputs and outputs.
2. Do not upgrade provider versions. Change provider requirements only when needed for a coding-
  standards fix, and leave provider selections and `.terraform.lock.hcl` files unchanged.

## Step 4 – Validate

Format the whole repository once from the root:

```bash
terraform fmt -recursive
```

Then initialise and validate the repository root **and** every directory under `examples/` and
`tests/` that contains Terraform files:

```bash
terraform -chdir=<dir> init -backend=false -input=false -lockfile=readonly
terraform -chdir=<dir> validate
```

- Fix errors caused by module updates and rerun until validation succeeds. Do not upgrade providers
  as part of validation.
- Never stage `.terraform/` folders, state files or plan files.
- If validation still fails after reasonable effort, do **not** open a pull request with broken
  code. Instead call `noop` explaining the blocking error, the attempted versions and the failing
  file(s).

## Step 5 – Coding standards

Apply the **Terraform Coding Standards** section to the changed repository. Perform the `fix`
actions in the same change set and collect all findings for the pull request body.

## Step 6 – Documentation

Update the documentation so it reflects the new state:

- `README.md` (root and examples): module usage snippets, `?ref=` / `version` values, and
  inputs/outputs tables when they changed. If the README
  contains `<!-- BEGIN_TF_DOCS -->` / `<!-- END_TF_DOCS -->` markers, keep the markers and update
  the content between them in the same terraform-docs format.
- `CHANGELOG.md` (only if it exists): add the changes under an `Unreleased` section following the
  existing format.

## Step 7 – Pull request

Commit all changes on a branch named `terraform-deps/update-dependencies` and create **one**
pull request with:

- **Title**: `Update Terraform modules (<n> module reference(s))`
- **Body**:
  1. **Summary** – one paragraph describing what was updated and why.
  2. **Module version changes** – table with `Module`, `From`, `To`, `Files`.
  3. **Breaking changes & migrations** – what was adapted for major upgrades (or "None").
  4. **Validation** – the commands you ran per directory and their result.
  5. **Coding standards review** – the table described in the coding standards section.
  6. **Documentation** – the documentation files that were updated.

Keep the change focused: do not refactor unrelated code, rename resources or change behaviour
beyond what the upgrade and the `fix` actions require.
