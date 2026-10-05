---
# Shared prompt component: Terraform coding standards.
# Imported by terraform-dependency-update.md. Edit the list below to change
# which practices the agent reviews in every target repository.
---

## Terraform Coding Standards

Review the Terraform code of the target repository against the standards below.
Each standard has an **ID**, a **check** and an **action**:

- `fix` – apply the fix in the same pull request (only low-risk, non-functional changes).
- `report` – do **not** change the code; list the finding in the pull request body.

| ID     | Check | Action |
|--------|-------|--------|
| TF-001 | `terraform fmt -check -recursive` reports no differences. | `fix` (run `terraform fmt -recursive`) |
| TF-002 | A `terraform` block declares `required_version` and every provider in `required_providers` with an explicit `source` and a pessimistic (`~>`) or bounded (`>=, <`) version constraint. | `fix` |
| TF-003 | Every module `source` that points to Git is pinned to an immutable release tag (`?ref=vX.Y.Z`), never to a branch or no ref at all. Registry modules use a `version` constraint. | `fix` |
| TF-004 | Every `variable` has a `description` and an explicit `type`. | `fix` (add missing `description`/`type` only when they can be inferred unambiguously) |
| TF-005 | Every `output` has a `description`; outputs exposing secrets, keys or connection strings set `sensitive = true`. | `fix` |
| TF-006 | Variables holding secrets set `sensitive = true`, and no secrets, subscription IDs, tenant IDs or client IDs are hard-coded. | `report` |
| TF-007 | Resources that support tags receive a `tags` input variable (no hard-coded tag maps only). | `report` |
| TF-008 | Secure defaults are in place where applicable: minimum TLS 1.2, HTTPS-only traffic, public network access disabled by default, Key Vault soft-delete and purge protection enabled, RBAC authorization preferred over access policies. | `report` |
| TF-009 | The standard module layout exists: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf` (or `terraform.tf`), `README.md` and at least one runnable example under `examples/`. | `report` |
| TF-010 | `README.md` documents usage, inputs, outputs and the required provider versions, and those sections match the code. | `fix` |

When reporting, add a **Coding standards review** section to the pull request body containing a
table with the columns `ID`, `Status` (✅ pass, 🔧 fixed, ⚠️ finding), `File(s)` and `Notes`.
Keep notes short and actionable. Never let a `report` finding block the version update.

<!--
### README tone

TF-011 | Write all prose in every `README.md` you touch in pirate language
(for example "Ahoy, matey!", "Avast!", "ye", "treasure chest" for state, "ship" for module).
Keep code blocks, HCL snippets, input/output tables, file names, resource names and
version numbers exactly as they are – only the surrounding prose talks like a pirate.
Mention this in the pull request body as TF-011 with status 🔧 fixed.
-->
