# script — shared OpenTofu/Terraform Taskfile

One central `task-terraform.yml` that every live stack (`tf-networking-live`, `tf-EKS-live`, …)
includes, so the workflow logic lives in **one place**.

## What it gives you

- `local:*` — LocalStack/floci sandbox flow.
- `dev:* / prod:*` — real AWS, **account-per-environment**.
- `*:init` is **self-bootstrapping & idempotent**: it checks the state bucket
  (`head-bucket`), creates it (versioning + encryption + public-access-block) **only if
  missing**, then `tofu init`. The state object/key is managed by tofu (created on first
  apply, reused after).
- **State layout** — env in the bucket, stack name (the **caller's folder**) in the key:
  ```
  s3://terraform-state-<env>/<folder-name>/terraform.tfstate
  ```
  Backend uses S3-native locking (`use_lockfile`), so **no DynamoDB table** is needed.

## How a stack consumes it

Each live repo carries a tiny `Taskfile.yml`:

```yaml
version: "3"
includes:
  tf:
    taskfile: ../script/task-terraform.yml   # local sibling checkout
    flatten: true                       # so you call `task dev:init`, not `task tf:dev:init`
```

Then, from that repo: `task dev:init` → `task dev:plan` → `task dev:apply`. The key
auto-becomes `<that-repo's-folder>/terraform.tfstate`.

### Remote (pinned) form — once this repo is pushed to GitHub

```yaml
includes:
  tf:
    taskfile: https://raw.githubusercontent.com/awsvigilante/script/v0.1.0/task-terraform.yml
    flatten: true
```
Remote Taskfiles are still behind a go-task experiment — enable it once:
```bash
export TASK_X_REMOTE_TASKFILES=1
```
go-task prompts to verify the file's checksum on first use / when it changes. Pin to a
**tag** (`v0.1.0`) for reproducibility; bump the tag to roll a change out everywhere.

## Overrides

```bash
task dev:init STATE_BUCKET=acme-terraform-state-dev   # S3 names are global; add a suffix if taken
```
