# Compute stack (CloudFormation)

Minimal Lambda on AWS Free Tier, provisioned with the **compute** CloudFormation stack.

## AWS CLI

**Recommended (CLI v2):** In PowerShell, run `winget install -e Amazon.AWSCLI` and accept the UAC prompt.

**Alternative (CLI v1 via Python, no admin):** `python -m pip install --user awscli` — ensure your Python `Scripts` folder is on `PATH`, then run `aws --version`.

Configure credentials once: `aws configure` (Access Key, Secret Key, region, output `json`).

## What gets created

| Resource | Purpose |
|----------|---------|
| `AWS::Logs::LogGroup` | Stores Lambda execution logs (7-day retention) |
| `AWS::IAM::Role` | Lets Lambda run and write to CloudWatch Logs |
| `AWS::Lambda::Function` | Your Python handler (`index.handler`) |

## Deploy with AWS CLI

```powershell
cd "C:\Projects\AWS_learn\Simple lambda"

aws cloudformation deploy `
  --template-file compute-stack.yaml `
  --stack-name simple-compute-dev `
  --capabilities CAPABILITY_IAM
```

## Test

```powershell
aws lambda invoke `
  --function-name simple-compute-dev-hello `
  --cli-binary-format raw-in-base64-out `
  --payload "{}" `
  out.json

Get-Content out.json
```

## Delete (stop charges / cleanup)

```powershell
aws cloudformation delete-stack --stack-name simple-compute-dev
```

## Deploy via AWS Console (no CLI)

1. Open **CloudFormation** in your region.
2. **Create stack** → **Upload a template file** → choose `compute-stack.yaml`.
3. Stack name: `simple-compute-dev` → Next → Next → Submit.
4. **Lambda** → open `simple-compute-dev-hello` → **Test** with `{}`.

## Deploy with GitHub Actions

Workflows live under `.github/workflows/`.

| File | Type | What it does |
|------|------|----------------|
| **`ci.yml`** | **CI** | Runs `cfn-lint` on `compute-stack.yaml` (pull requests and pushes to `main`) — no AWS deploy |
| **`cd.yml`** | **CD** | `aws cloudformation deploy` to **`simple-compute-dev`** |

**CD triggers:** push to `main` when the template or `cd.yml` changes (uses **OIDC**), or manual **Run workflow**.

**Manual CD auth:** choose **oidc** (needs `AWS_ROLE_ARN`) or **access-keys** (needs `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`).

Change `AWS_REGION` in `cd.yml` `env` if you do not use `ap-south-1`.

Typical flow: open a PR → **CI** validates → merge to `main` → **CD** deploys.

### Option A — OIDC (recommended for CD on push)

1. Push this repo to GitHub.
2. In AWS IAM, create an OIDC identity provider for `token.actions.githubusercontent.com` (if you do not have one).
3. Create an IAM role GitHub can assume (trust policy: your org/repo, e.g. `repo:YOUR_USER/YOUR_REPO:*`).
4. Attach permissions to deploy CloudFormation and create Lambda/IAM/Logs (e.g. `AdministratorAccess` for learning only).
5. GitHub repo → **Settings** → **Secrets and variables** → **Actions** → secret **`AWS_ROLE_ARN`** = role ARN.
6. **Actions** → **CD** → **Run workflow**, or merge to `main` after **CI** passes.

### Option B — Access keys (manual CD only)

1. GitHub secrets: **`AWS_ACCESS_KEY_ID`**, **`AWS_SECRET_ACCESS_KEY`**.
2. **Actions** → **CD** → **Run workflow** → auth **access-keys**.

Do not commit access keys to the repo.
