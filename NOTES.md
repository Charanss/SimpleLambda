# Project notes

Learning log for [SimpleLambda](https://github.com/Charanss/SimpleLambda).  
**Default region:** `ap-south-1` (Mumbai)  
**CloudFormation templates:** `cfn_templates/` — **`master-stack.yaml`**, `compute-stack.yaml`, `persistent-stack.yaml`, **`network-stack.yaml`** (CloudFront + edge Lambda, deploy **us-east-1**)

---

## Master stack (recommended)

**Stack name:** `simple-master-dev`  
**Template:** `cfn_templates/master-stack.yaml`

**What we did**

- **Network stack** (`network-stack.yaml`): CloudFront + **Lambda@Edge** in **one template**; deploy **`simple-network-dev`** in **us-east-1** after master.
- Security headers on **viewer-response** (CSP, HSTS, X-Content-Type-Options). **Not** in compute stack.
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
**Templates:** `cfn_templates/network-stack.yaml` only (edge Lambda + CloudFront)  
**Stack name:** **`simple-network-dev`** in **us-east-1**

**What we did**

- **Persistent stack** = S3 only (no CloudFront).
- **Network stack** = CloudFront origin → S3 website; **Lambda@Edge** on **viewer-response** adds:
  - `Content-Security-Policy`
  - `Strict-Transport-Security` (HTTPS transport security)
  - `X-Content-Type-Options: nosniff`
- Edge function is **not** in compute stack (compute = regional Lambda API).

**How to verify**

```powershell
curl.exe -sS -I "https://YOUR_DIST.cloudfront.net/" | findstr /i "content-security-policy strict-transport x-content-type"
```

**Notes**

- Lambda@Edge must be deployed via **network-stack** in **us-east-1** (AWS requirement).
- Direct S3 URLs still have **no** edge headers.

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
