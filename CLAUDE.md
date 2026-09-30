# project-moonshard

## What this repo is

This repository holds **code only** for ML/SFT experiments run by Flagship
Pioneering. The code here is executed remotely on **Modal** (serverless
GPU/CPU compute — `modal run` / `modal deploy`), not run locally against
local data.

## What does NOT belong in this repo

- No datasets, raw inputs, or experiment outputs of any kind.
- No model weights or checkpoints.
- No credentials, API keys, or tokens (Modal, AWS, HuggingFace, etc.) —
  even as defaults, examples, or "temporary" test values.
- No large binary files in general.

All of the above lives in a single AWS S3 bucket: **`intellexi`**
(region `us-east-1`). If you're writing code that reads input data or
writes results, it should read from / write to that bucket — via a Modal
`CloudBucketMount`, or via `boto3`/the AWS CLI using the `intellexi` AWS
SSO profile already configured on the developer's machine — never to a
path inside this repo, and never assumed to persist on local/Modal
container disk between runs.

Do not create local data directories in this repo (e.g. `data/`,
`outputs/`, `checkpoints/`) as a convenience, even temporarily — if you
need scratch space while iterating, use `/tmp` or a Modal Volume for
transient/cache use only, not this repo.

## Git workflow expectations

- Local commits on a feature/working branch: fine to do freely.
- `git push`, force-push, branch deletion, or history rewriting: do not
  do this without the user explicitly confirming first, every time — not
  a one-time approval.
- Never commit a `.env` file, `~/.aws/`, `~/.modal.toml`, or anything
  else that could contain credentials. Check `.gitignore` covers these
  before adding new file types that might carry secrets.

## Modal conventions to follow

- Use `<app>-dev` / `<app>-prod` Modal environments, not the default
  environment, for anything beyond a one-off `modal run` test.
- Never deploy to a `-prod` Modal environment from a local machine —
  production deploys go through CI with a service-user token.
- Reference Modal secrets via `modal.Secret.from_name(...)`. Never
  hardcode a token or key inline in code, even for a "quick test."
- Tag Modal apps with a chargeback-relevant prefix if/when this project
  is deployed persistently (ask the user what prefix applies before
  assuming one).

## Modal secrets already configured (in `intellexi-dev` environment)

These exist in Modal's secret store already — do not create duplicates,
and never ask the user to paste the raw values into chat or into a file.
Reference by name only:

| Secret name          | Provides                          | Used for                                   |
| --------------------- | ---------------------------------- | ------------------------------------------- |
| `aws-intellexi`       | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Scoped IAM user (`intellexi-modal`), read/write access to the `intellexi` S3 bucket only — not account-wide. Used via `modal.CloudBucketMount("intellexi", secret=modal.Secret.from_name("aws-intellexi"))`. |
| `huggingface-token`   | `HF_TOKEN`                         | Downloading models/datasets from the HuggingFace Hub, including gated repos the user has accepted access to. Reference via `modal.Secret.from_name("huggingface-token")`; the `huggingface_hub`/`transformers` libraries pick up `HF_TOKEN` automatically. |
| `wandb-api-key`       | `WANDB_API_KEY`                    | Experiment tracking. Reference via `modal.Secret.from_name("wandb-api-key")`; `wandb.init()` picks up `WANDB_API_KEY` automatically — no explicit `wandb.login()` call needed inside functions. |

All three were validated end-to-end (list/write/read for S3;
`whoami()`/`login(verify=True)` for HF/W&B) from a Modal function in
`intellexi-dev` on 2026-09-29. If any of these secrets stop working,
that's a sign the credential was rotated/revoked outside this repo —
flag it to the user rather than assuming you should create a new one.

W&B project convention: use `intellexi-sft` (or a clearly-related name)
as the `wandb.init(project=...)` value so runs group sensibly, rather
than letting runs fall into a default/unnamed project.

## If you're an agent picking this up cold

1. Read this file fully before writing or running anything.
2. Assume any data you need is in the `intellexi` S3 bucket, not in this
   repo and not already on disk wherever you're executing.
3. If a task seems to require storing data in this repo, or pushing/
   force-pushing/deleting branches, stop and ask the user first — these
   are intentional guardrails, not oversights.
