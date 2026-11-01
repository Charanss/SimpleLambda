# SimpleLambda

CloudFormation learning project: regional Lambda, S3 static site, CloudFront, Lambda@Edge security headers, and WAF path rules.

See **[NOTES.md](./NOTES.md)** for a feature-by-feature log.

## Deploy to AWS (GitHub Actions only)

**All infrastructure and site updates deploy through GitHub Actions — not from local `aws cloudformation deploy` or the AWS Console.**

| Workflow | File | Purpose |
|----------|------|---------|
| **CI** | [`.github/workflows/ci.yml`](.github/workflows/ci.yml) | `cfn-lint` on `cfn_templates/` (no AWS changes) |
| **CD** | [`.github/workflows/cd.yml`](.github/workflows/cd.yml) | Full deploy: S3 templates, edge Lambda, WAF, master stack, `index.html`, CloudFront invalidation |

**When CD runs**

- **CI** workflow on push to **`main`**: after **cfn-lint** passes, the **deploy** job runs **`cd.yml`**
- Or **Actions → CD → Run workflow** (manual only; same deploy steps)

**GitHub secrets (CD)**

- **access-keys (auto deploy after CI):** `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- **oidc (manual runs):** `AWS_ROLE_ARN`

Typical flow: PR → **CI** (lint only) → merge to **main** → **CI** (lint then deploy in one run).

Default region for master/data: **`ap-south-1`**. Lambda@Edge + WAF web ACL stacks: **`us-east-1`**.

## What gets created

| Stack / nested | Region | Purpose |
|----------------|--------|---------|
| `simple-master-dev` | ap-south-1 | Nested compute, persistent, network (CloudFront) |
| `simple-compute-dev-edge` | us-east-1 | Lambda@Edge (`applySecurityHeaders`) |
| `simple-waf-dev` | us-east-1 | WAF web ACL (attached to CloudFront via `WebACLId`) |

**WAF (HTTP 400)** blocks URI paths matching admin probes and backup/archive patterns (e.g. `backup`, `*.bak`, `*.old`, `*.zip`).

## Verify (read-only — use AWS CLI or browser)

**CloudFront URL** (from master stack **Outputs** in **ap-south-1**):

```powershell
$URL = aws cloudformation describe-stacks --stack-name simple-master-dev --region ap-south-1 `
  --query "Stacks[0].Outputs[?OutputKey=='CloudFrontURL'].OutputValue" --output text
curl.exe -sS -I "$URL/index.html" | findstr /i "content-security-policy strict-transport x-content-type"
curl.exe -sS -I "$URL/backup" | findstr /i "400"
curl.exe -sS -I "$URL/config.bak" | findstr /i "400"
```

**Regional hello Lambda** (invoke only — not part of CD site):

```powershell
aws lambda invoke --function-name simple-compute-dev-hello --region ap-south-1 `
  --cli-binary-format raw-in-base64-out --payload "{}" out.json
Get-Content out.json
```

Direct S3 website URLs do **not** show CSP or WAF rules — test **CloudFront** only.

## AWS CLI on your PC

Install and `aws configure` only for **verification** and learning (invoke Lambda, read stack outputs). **Do not** use it to create or update CloudFormation stacks for this repo; use **CD** instead.

```powershell
winget install -e Amazon.AWSCLI
aws configure
```

## Cleanup

Remove stacks from the AWS Console or CLI when finished experimenting (master, edge, WAF) to avoid ongoing WAF/CloudFront charges.
