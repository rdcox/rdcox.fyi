# rdcox.fyi Blog Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Stand up a Hugo blog with PaperMod theme, deployed to AWS S3+CloudFront via GitHub Actions, accessible at rdcox.fyi.

**Architecture:** Hugo static site generator with PaperMod theme, hosted on S3 behind CloudFront CDN. Route 53 points rdcox.fyi to CloudFront. GitHub Actions builds and deploys on push to main.

**Tech Stack:** Hugo, PaperMod theme, AWS (S3, CloudFront, ACM, Route 53, IAM), GitHub Actions

---

### Task 1: Initialize Hugo Project

**Files:**
- Create: `hugo.yaml`
- Create: `archetypes/default.md`

**Step 1: Install Hugo (if not already installed)**

Run: `brew install hugo`
Expected: Hugo installed successfully

**Step 2: Verify Hugo installation**

Run: `hugo version`
Expected: Output showing hugo version (v0.1XX.X or similar)

**Step 3: Create new Hugo site in current directory**

Since we already have files in this directory (docs/plans), we need to scaffold Hugo around them:

Run: `hugo new site . --force`
Expected: Hugo scaffolds project files without overwriting existing files

**Step 4: Verify project structure**

Run: `ls -la`
Expected: Directories including `archetypes/`, `content/`, `layouts/`, `static/`, `themes/`, and `hugo.toml`

**Step 5: Convert hugo.toml to hugo.yaml (PaperMod docs use YAML)**

Delete the generated `hugo.toml` and create `hugo.yaml` with minimal config:

```yaml
baseURL: "https://rdcox.fyi/"
languageCode: "en-us"
title: "rdcox.fyi"
paginate: 10
```

**Step 6: Commit**

```bash
git add -A
git commit -m "feat: initialize Hugo project"
```

---

### Task 2: Install PaperMod Theme

**Files:**
- Modify: `hugo.yaml`
- Create: `.gitmodules`

**Step 1: Add PaperMod as a git submodule**

Run: `git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod`
Expected: Theme cloned into `themes/PaperMod/`

**Step 2: Update hugo.yaml with theme and PaperMod configuration**

Replace the contents of `hugo.yaml` with:

```yaml
baseURL: "https://rdcox.fyi/"
languageCode: "en-us"
title: "rdcox.fyi"
paginate: 10
theme: PaperMod

enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false

minify:
  disableXML: true
  minifyOutput: true

params:
  env: production
  title: "rdcox.fyi"
  description: "Thoughts and work by rdcox"
  author: "rdcox"
  DateFormat: "January 2, 2006"
  defaultTheme: auto
  disableThemeToggle: false
  ShowReadingTime: true
  ShowShareButtons: false
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowWordCount: false
  ShowRssButtonInSectionTermList: true
  UseHugoToc: true
  comments: false

  homeInfoParams:
    Title: "rdcox.fyi"
    Content: "Thoughts and work."

  socialIcons:
    - name: github
      url: "https://github.com/rdcox"

menu:
  main:
    - identifier: posts
      name: Posts
      url: /posts/
      weight: 10
    - identifier: projects
      name: Projects
      url: /projects/
      weight: 20
    - identifier: tags
      name: Tags
      url: /tags/
      weight: 30

markup:
  highlight:
    noClasses: false

pygmentsUseClasses: true
```

**Step 3: Verify the site builds and serves locally**

Run: `hugo server -D`
Expected: Server starts at http://localhost:1313/, site renders with PaperMod theme

**Step 4: Stop the server (Ctrl+C) and commit**

```bash
git add -A
git commit -m "feat: add PaperMod theme and site configuration"
```

---

### Task 3: Create Content Structure and First Post

**Files:**
- Create: `content/posts/_index.md`
- Create: `content/projects/_index.md`
- Create: `content/posts/hello-world.md`

**Step 1: Create the posts section index**

Create `content/posts/_index.md`:

```markdown
---
title: "Posts"
---
```

**Step 2: Create the projects section index**

Create `content/projects/_index.md`:

```markdown
---
title: "Projects"
---
```

**Step 3: Create a first post**

Create `content/posts/hello-world.md`:

```markdown
---
title: "Hello, World"
date: 2026-03-07
draft: false
tags: ["meta"]
summary: "First post on rdcox.fyi."
---

This is the first post on rdcox.fyi. More to come.
```

**Step 4: Verify the site builds with the new content**

Run: `hugo server`
Expected: Site serves at localhost:1313, "Hello, World" post appears on homepage and under /posts/

**Step 5: Stop the server and commit**

```bash
git add content/
git commit -m "feat: add content structure and first post"
```

---

### Task 4: Create S3 Bucket for Static Hosting

**Prerequisite:** AWS CLI installed and configured with credentials that can create S3 buckets.

**Step 1: Verify AWS CLI is configured**

Run: `aws sts get-caller-identity`
Expected: JSON output with your AWS account info

**Step 2: Create the S3 bucket**

Run: `aws s3 mb s3://rdcox.fyi --region us-east-1`
Expected: `make_bucket: rdcox.fyi`

**Step 3: Configure the bucket for static website hosting**

Run: `aws s3 website s3://rdcox.fyi --index-document index.html --error-document 404.html`
Expected: No output (success)

**Step 4: Set bucket policy to allow CloudFront access (we'll use OAC)**

Create a temporary file `/tmp/bucket-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::rdcox.fyi/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "CLOUDFRONT_DISTRIBUTION_ARN_PLACEHOLDER"
        }
      }
    }
  ]
}
```

**Note:** We'll update the bucket policy after creating the CloudFront distribution in Task 5. Skip applying it for now.

**Step 5: Commit (no files changed in repo — this is infrastructure)**

No commit needed for this task.

---

### Task 5: Request ACM Certificate

**Step 1: Request a certificate for rdcox.fyi in us-east-1**

Run:
```bash
aws acm request-certificate \
  --domain-name rdcox.fyi \
  --validation-method DNS \
  --region us-east-1
```
Expected: JSON output with `CertificateArn`

**Step 2: Save the certificate ARN — you'll need it for CloudFront**

Run:
```bash
aws acm describe-certificate \
  --certificate-arn <ARN_FROM_STEP_1> \
  --region us-east-1 \
  --query 'Certificate.DomainValidationOptions'
```
Expected: JSON with DNS validation CNAME record details

**Step 3: Create the DNS validation record in Route 53**

Get your hosted zone ID:
```bash
aws route53 list-hosted-zones-by-name --dns-name rdcox.fyi --query 'HostedZones[0].Id' --output text
```

Then create the validation record:
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "<CNAME_NAME_FROM_STEP_2>",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "<CNAME_VALUE_FROM_STEP_2>"}]
      }
    }]
  }'
```
Expected: JSON with ChangeInfo status PENDING

**Step 4: Wait for certificate validation**

Run: `aws acm wait certificate-validated --certificate-arn <ARN> --region us-east-1`
Expected: Command completes (may take a few minutes)

---

### Task 6: Create CloudFront Distribution

**Step 1: Create a CloudFront distribution with OAC**

First, create an Origin Access Control:
```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "rdcox-fyi-oac",
    "Description": "OAC for rdcox.fyi S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```
Expected: JSON with OAC Id

**Step 2: Create the CloudFront distribution**

Create `/tmp/cf-distribution.json`:
```json
{
  "CallerReference": "rdcox-fyi-2026-03-07",
  "Aliases": {
    "Quantity": 1,
    "Items": ["rdcox.fyi"]
  },
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3-rdcox-fyi",
        "DomainName": "rdcox.fyi.s3.us-east-1.amazonaws.com",
        "OriginAccessControlId": "<OAC_ID_FROM_STEP_1>",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-rdcox-fyi",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [
      {
        "ErrorCode": 404,
        "ResponsePagePath": "/404.html",
        "ResponseCode": "404",
        "ErrorCachingMinTTL": 300
      }
    ]
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "<ACM_CERT_ARN>",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "Enabled": true,
  "Comment": "rdcox.fyi blog"
}
```

Run:
```bash
aws cloudfront create-distribution --distribution-config file:///tmp/cf-distribution.json
```
Expected: JSON with Distribution Id, ARN, and DomainName (e.g., d1234abcdef.cloudfront.net)

**Step 3: Update the S3 bucket policy with the CloudFront distribution ARN**

Update `/tmp/bucket-policy.json` replacing `CLOUDFRONT_DISTRIBUTION_ARN_PLACEHOLDER` with the actual ARN from Step 2, then apply:

```bash
aws s3api put-bucket-policy --bucket rdcox.fyi --policy file:///tmp/bucket-policy.json
```
Expected: No output (success)

---

### Task 7: Configure Route 53 DNS

**Step 1: Create an A record alias pointing to CloudFront**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "rdcox.fyi",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "<CLOUDFRONT_DOMAIN_NAME>",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

Note: `Z2FDTNDATAQYW2` is the fixed hosted zone ID for all CloudFront distributions.

Expected: JSON with ChangeInfo status PENDING

**Step 2: Verify DNS resolution (may take a few minutes)**

Run: `dig rdcox.fyi`
Expected: A record pointing to CloudFront IPs

---

### Task 8: Do an Initial Manual Deploy

**Step 1: Build the site**

Run: `hugo --minify`
Expected: Output showing pages built, files written to `public/`

**Step 2: Sync to S3**

Run: `aws s3 sync public/ s3://rdcox.fyi --delete`
Expected: Files uploaded to S3

**Step 3: Invalidate CloudFront cache**

Run:
```bash
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```
Expected: JSON with Invalidation Id

**Step 4: Verify the site is live**

Run: `curl -I https://rdcox.fyi`
Expected: HTTP 200, content served via CloudFront

**Step 5: Add `public/` to .gitignore and commit**

Create `.gitignore`:
```
public/
resources/_gen/
.hugo_build.lock
```

```bash
git add .gitignore
git commit -m "chore: add .gitignore for Hugo build artifacts"
```

---

### Task 9: Create GitHub Repository

**Step 1: Create a new GitHub repo**

Run: `gh repo create rdcox/rdcox.fyi --private --source=. --remote=origin`
Expected: Repository created on GitHub

**Step 2: Push to GitHub**

Run: `git push -u origin main`
Expected: All commits pushed to remote

---

### Task 10: Create IAM User for GitHub Actions

**Step 1: Create a dedicated IAM user**

Run: `aws iam create-user --user-name github-actions-rdcox-fyi`
Expected: JSON with User details

**Step 2: Create and attach an inline policy**

Create `/tmp/deploy-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::rdcox.fyi",
        "arn:aws:s3:::rdcox.fyi/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "<CLOUDFRONT_DISTRIBUTION_ARN>"
    }
  ]
}
```

Run:
```bash
aws iam put-user-policy \
  --user-name github-actions-rdcox-fyi \
  --policy-name deploy-to-s3 \
  --policy-document file:///tmp/deploy-policy.json
```
Expected: No output (success)

**Step 3: Create access keys**

Run: `aws iam create-access-key --user-name github-actions-rdcox-fyi`
Expected: JSON with AccessKeyId and SecretAccessKey — **save these securely**

**Step 4: Add secrets to GitHub repo**

```bash
gh secret set AWS_ACCESS_KEY_ID --body "<ACCESS_KEY_ID>"
gh secret set AWS_SECRET_ACCESS_KEY --body "<SECRET_ACCESS_KEY>"
gh secret set CLOUDFRONT_DISTRIBUTION_ID --body "<DISTRIBUTION_ID>"
```
Expected: Each secret set successfully

---

### Task 11: Create GitHub Actions Deploy Workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

**Step 1: Create the workflow file**

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to S3

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "latest"
          extended: true

      - name: Build
        run: hugo --minify

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to S3
        run: aws s3 sync public/ s3://rdcox.fyi --delete

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

**Step 2: Verify the workflow file is valid YAML**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/deploy.yml'))"`
Expected: No error

**Step 3: Commit and push**

```bash
git add .github/
git commit -m "feat: add GitHub Actions deploy workflow"
git push
```
Expected: Push triggers the workflow; verify on GitHub Actions tab

**Step 4: Verify the deploy workflow ran successfully**

Run: `gh run list --limit 1`
Expected: Most recent run shows ✓ completed

---

### Task 12: Final Verification

**Step 1: Verify the site loads at https://rdcox.fyi**

Run: `curl -I https://rdcox.fyi`
Expected: HTTP 200 with CloudFront headers

**Step 2: Test creating a new post and deploying**

Create a test post:
```bash
hugo new content posts/test-deploy.md
```

Edit it to set `draft: false` and add some content, then:
```bash
git add content/posts/test-deploy.md
git commit -m "test: verify deploy pipeline"
git push
```

**Step 3: Wait for deploy and verify**

Run: `gh run watch`
Expected: Workflow completes successfully

Verify: `curl https://rdcox.fyi/posts/test-deploy/`
Expected: HTML content of the test post

**Step 4: Clean up test post (optional)**

```bash
git rm content/posts/test-deploy.md
git commit -m "chore: remove test post"
git push
```
