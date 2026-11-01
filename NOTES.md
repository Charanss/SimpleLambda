# Project notes

Learning log for [SimpleLambda](https://github.com/Charanss/SimpleLambda).  
**Default region:** `ap-south-1` (Mumbai)

When you add a feature, copy the **New feature template** at the bottom into a new dated section above it.

---

## Compute stack (Lambda)

**Date:** Initial setup  
**Stack name:** `simple-compute-dev`  
**Template:** `compute-stack.yaml`

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
- Stack names from workflow `env`: `COMPUTE_STACK_NAME`, `STORAGE_STACK_NAME`.
- AWS account for deploy = account that owns the access keys (no separate account ID in GitHub).

**How to verify**

- Actions → **CI** on PR/push; **CD** on push to `main` or manual run.

**Notes**

- Rotate keys if leaked; never commit secrets.
- IAM user needs permissions for every service the stacks create (Lambda, S3, CloudFront, etc.).

---

## Storage stack (S3 static site)

**Date:** After compute  
**Stack name:** `simple-storage-dev`  
**Template:** `storage-stack.yaml`  
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

## CloudFront + CSP response header

**Date:** Edge / security header feature  
**Stack:** same `simple-storage-dev` (not a separate “network stack”)  
**Resources:** `SiteCspHeadersPolicy`, `SiteDistribution`

**What we did**

- CloudFront distribution in front of S3 **website** origin.
- **ResponseHeadersPolicy** sets HTTP header:  
  `Content-Security-Policy: default-src 'self'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; base-uri 'self'`
- Outputs: **CloudFrontURL** (preferred browser URL), **CloudFrontDistributionId**.
- CD runs **CloudFront invalidation** after `index.html` upload.

**How to verify**

```powershell
curl.exe -sS -I "https://YOUR_DIST.cloudfront.net/" | findstr /i content-security-policy
```

**Notes**

- First distribution create can take several minutes.
- CSP on edge (CloudFront), not VPC “network stack”.
- `'unsafe-inline'` needed for current inline script/style; tighten later with nonces if you refactor HTML.

---

## New feature template

Copy this block for each new feature:

```markdown
## [Feature name]

**Date:** YYYY-MM-DD  
**Stack / service:** (e.g. `simple-storage-dev`, new template name)  
**Files changed:** (templates, workflows, static assets)

**What we did**

- 

**How to verify**

- 

**Notes**

- 
```
