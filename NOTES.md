# Project notes

Learning log for [SimpleLambda](https://github.com/Charanss/SimpleLambda).  
**Default region:** `ap-south-1` (Mumbai)  
**CloudFormation templates:** `cfn_templates/` — **`master-stack.yaml`** nests compute, persistent, and **network**; **`waf-stack.yaml`** + edge compute in **us-east-1**. **Deploy only via GitHub Actions CD.**

---

## Master stack (recommended)

**Stack name:** `simple-master-dev`  
**Template:** `cfn_templates/master-stack.yaml`

**What we did**

- **Compute stack** (`compute-stack.yaml`): all Lambdas — regional hello (master / ap-south-1) and **edge-security** (`simple-compute-dev-edge` in **us-east-1**).
- **Network stack** nested in master: CloudFront + **`EdgeLambdaVersionArn`** from **`simple-compute-dev-edge`** (us-east-1).
- All outputs (Lambda, bucket, **CloudFrontURL**, etc.) on the **master** stack.

**How to verify**

- CloudFormation → **`simple-master-dev`** → **Outputs** (region **ap-south-1**).
- Nested stacks: **ComputeStack**, **PersistentStack**, **NetworkStack** under **`simple-master-dev`**.

**Notes**

- Nested templates must be in S3 (`TemplateURL`); master is deployed from the repo file.

---

## Compute stack (Lambda)

**Date:** Initial setup  
**Stack name:** `simple-compute-dev`  
**Template:** `cfn_templates/compute-stack.yaml`

**What we did**

- CloudFormation **compute** stack: Lambda + IAM execution role + CloudWatch log group.
- Lambda code inline in template (`ZipFile`); handler `index.handler`, Python 3.12.
- Function name pattern: `{StackName}-hello` → `simple-compute-dev-hello`.
- Deploy requires `--capabilities CAPABILITY_IAM` (role created in stack).

**How to verify**

- `aws lambda invoke --function-name simple-compute-dev-hello --region ap-south-1 --payload "{}" out.json`
- Console: Lambda → Test with `{}`.

**Notes**

- IAM **role** is in the stack (`HelloLambdaRole`); not the same as GitHub deploy **user** credentials.

---

## GitHub Actions (CI / CD)

**Date:** After compute stack  
**Files:** `.github/workflows/ci.yml`, `.github/workflows/cd.yml`

**What we did**

- **CI:** `cfn-lint` on template changes (no AWS deploy).
- **CD:** `aws cloudformation deploy` using repo secrets (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`); manual run auth default `access-keys`.
- Stack names from workflow `env`: `MASTER_STACK_NAME`, `COMPUTE_RESOURCE_PREFIX`, `PERSISTENT_RESOURCE_PREFIX`.
- AWS account for deploy = account that owns the access keys (no separate account ID in GitHub).

**How to verify**

- Actions → **CI** on PR/push; **CD** on push to `main` or manual run.

**Notes**

- Rotate keys if leaked; never commit secrets.
- IAM user needs permissions for every service the stacks create (Lambda, S3, CloudFront, etc.).

---

## Persistent stack (S3 static site)

**Date:** After compute  
**Stack name:** `simple-persistent-dev` (standalone) or via master  
**Template:** `cfn_templates/persistent-stack.yaml`  
**Content:** `static/index.html` (inline `<script>`)

**What we did**

- S3 bucket: name `{StackName}-{AccountId}`.
- Static website hosting + bucket policy **public `s3:GetObject`** for demo HTML.
- CD uploads `static/index.html` after stack deploy (`aws s3 cp`).
- Direct S3 website URL in output **WebsiteURL**; object URL also works when public.

**How to verify**

- CloudFormation outputs → **WebsiteURL** or S3 object URL.
- `curl -I` on object URL (no CSP header on plain S3).

**Notes**

- S3 alone does not send **Content-Security-Policy** HTTP headers; use CloudFront for that.

---

## Network stack (CloudFront + Lambda@Edge headers)

**Date:** Edge security headers  
**Template:** `cfn_templates/network-stack.yaml` — nested as **NetworkStack** under **`simple-master-dev`** (**ap-south-1**, same region as compute + persistent)  
**Edge Lambda stack:** **`simple-compute-dev-edge`** in **us-east-1** only (AWS requirement)

**What we did**

- **Persistent stack** = S3 only (no CloudFront).
- **Network stack** (nested) = CloudFront origin → S3 website; **Lambda@Edge** on **viewer-request** (pass-through) and **origin-response** (adds headers):
  - `Content-Security-Policy`
  - `Strict-Transport-Security` (HTTPS transport security)
  - `X-Content-Type-Options: nosniff`
- Edge **function** is deployed via **compute-stack** in **us-east-1**; master passes **`EdgeLambdaVersionArn`** into nested network.

**How to verify**

```powershell
$URL = aws cloudformation describe-stacks --stack-name simple-master-dev --region ap-south-1 `
  --query "Stacks[0].Outputs[?OutputKey=='CloudFrontURL'].OutputValue" --output text
curl.exe -sS -I "$URL" | findstr /i "content-security-policy strict-transport x-content-type"
```

**Notes**

- **CloudFront stack** lives in **ap-south-1** with the master; only **Lambda@Edge publish** must use **us-east-1**.
- Direct S3 URLs still have **no** edge headers.

---

## WAF stack (path blocking, HTTP 400)

**Date:** 2026-11-01  
**Template:** `cfn_templates/waf-stack.yaml`  
**Stack name:** **`simple-waf-dev`** in **us-east-1** (web ACL attached on CloudFront via master → network **`WebACLId`**)

**What we did**

- **BlockAdminProbePaths:** `/admin`, `/secret`, `/.env`, `/wp-admin`, etc.
- **BlockBackupArchivePaths:** `backup` paths and extensions **`.bak`**, **`.old`**, **`.zip`**, **`.backup`**
- Block action returns **HTTP 400** with plain body `Bad Request`
- **Deploy policy:** AWS changes only via **GitHub Actions CD** (see README)

**How to verify**

```powershell
$URL = aws cloudformation describe-stacks --stack-name simple-master-dev --region ap-south-1 `
  --query "Stacks[0].Outputs[?OutputKey=='CloudFrontURL'].OutputValue" --output text
curl.exe -sS -I "$URL/backup"
curl.exe -sS -I "$URL/archive.zip"
curl.exe -sS -I "$URL/index.html"
```

**Notes**

- WAF is billed separately from Free Tier; delete **`simple-waf-dev`** when not needed.
- Rules apply on **CloudFront** only, not direct S3 URLs.

---

## CloudFront + CSP (superseded by network stack)

**Date:** Earlier approach  
**Replaced by:** Response headers policy moved to **Lambda@Edge** in network stack (see above).

---

## New feature template

Copy this block for each new feature:

```markdown
## [Feature name]

**Date:** YYYY-MM-DD  
**Stack / service:** (e.g. `simple-persistent-dev`, new template name)  
**Files changed:** (templates, workflows, static assets)

**What we did**

- 

**How to verify**

- 

**Notes**

- 
```
