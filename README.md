# Compute stack (CloudFormation)

Minimal Lambda on AWS Free Tier, provisioned with the **compute** CloudFormation stack.

See **[NOTES.md](./NOTES.md)** for a feature-by-feature learning log (add new sections there when you extend the project).

Templates live in **`cfn_templates/`**. **GitHub CD** deploys edge Lambda in **us-east-1**, then the **master stack** (`simple-master-dev`): compute + persistent (S3) + network (CloudFront + edge headers).

## AWS CLI

**Recommended (CLI v2):** In PowerShell, run `winget install -e Amazon.AWSCLI` and accept the UAC prompt.

**Alternative (CLI v1 via Python, no admin):** `python -m pip install --user awscli` — ensure your Python `Scripts` folder is on `PATH`, then run `aws --version`.

Configure credentials once: `aws configure` (Access Key, Secret Key, region, output `json`).

## What gets created

| Resource | Purpose |
|----------|---------|
| `AWS::Logs::LogGroup` | Stores Lambda execution logs (7-day retention) |
| `AWS::IAM::Role` | Lets Lambda run and write to CloudWatch Logs |
| `AWS::Lambda::Function` | Python hello-world (`EnableRegionalApp`) |
| `AWS::Lambda::Function` | Lambda@Edge security headers (`EnableEdgeFunction`, deploy in **us-east-1**) |

## Persistent stack (`cfn_templates/persistent-stack.yaml`)

| Resource | Purpose |
|----------|---------|
| `AWS::S3::Bucket` | Static website bucket (data) |
| `static/index.html` | Uploaded by CD / CLI |

## Network stack (`cfn_templates/network-stack.yaml`)

| Resource | Purpose |
|----------|---------|
| `AWS::CloudFront::Distribution` | CDN; associates **edge Lambda from compute stack** |

```powershell
curl.exe -sS -I "https://YOUR_DIST.cloudfront.net/" | findstr /i "content-security-policy strict-transport-security x-content-type-options"
```

## Deploy with AWS CLI

**Master stack (compute + persistent):**

```powershell
cd "C:\Projects\AWS_learn\Simple lambda"
$ACCOUNT = aws sts get-caller-identity --query Account --output text
$TEMPLATES_BUCKET = "simple-lambda-cfn-$ACCOUNT"
aws s3 mb "s3://$TEMPLATES_BUCKET" --region ap-south-1 2>$null
aws s3 cp cfn_templates/compute-stack.yaml "s3://$TEMPLATES_BUCKET/templates/compute-stack.yaml" --region ap-south-1
aws s3 cp cfn_templates/persistent-stack.yaml "s3://$TEMPLATES_BUCKET/templates/persistent-stack.yaml" --region ap-south-1
aws cloudformation deploy `
  --template-file cfn_templates/master-stack.yaml `
  --stack-name simple-master-dev `
  --region ap-south-1 `
  --capabilities CAPABILITY_IAM `
  --parameter-overrides TemplateS3Bucket=$TEMPLATES_BUCKET TemplateS3Prefix=templates/

$MASTER = simple-master-dev
$BUCKET = aws cloudformation describe-stacks --stack-name $MASTER --region ap-south-1 `
  --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue" --output text
aws s3 cp static/index.html "s3://$BUCKET/index.html" --content-type "text/html; charset=utf-8" --region ap-south-1
$DIST_ID = aws cloudformation describe-stacks --stack-name $MASTER --region ap-south-1 `
  --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" --output text
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/index.html" "/"
```

**Optional — deploy child stacks alone:**

```powershell
aws cloudformation deploy `
  --template-file cfn_templates/compute-stack.yaml `
  --stack-name simple-compute-dev `
  --capabilities CAPABILITY_IAM

aws cloudformation deploy `
  --template-file cfn_templates/persistent-stack.yaml `
  --stack-name simple-persistent-dev
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
2. **Create stack** → **Upload a template file** → choose `cfn_templates/compute-stack.yaml`.
3. Stack name: `simple-compute-dev` → Next → Next → Submit.
4. **Lambda** → open `simple-compute-dev-hello` → **Test** with `{}`.

## Deploy with GitHub Actions

Workflows live under `.github/workflows/`.

| File | Type | What it does |
|------|------|----------------|
| **`ci.yml`** | **CI** | Runs `cfn-lint` on templates under **`cfn_templates/`** — no AWS deploy |
| **`cd.yml`** | **CD** | Upload child templates to S3, deploy **`simple-master-dev`**, upload `index.html`, invalidate CloudFront |

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
