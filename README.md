# GitHub Actions + OIDC + AWS Deployment Documentation

## 1. Overview

This document explains how GitHub Actions can securely authenticate with AWS using **OpenID Connect (OIDC)** and deploy an application to AWS.

The main focus of this documentation is understanding:

- GitHub Actions
- GitHub OIDC
- AWS STS
- IAM Trust Policy
- IAM Permissions Policy
- Temporary AWS Credentials
- Amazon S3
- Amazon CloudFront

---

## 2. Overall Flow

```text
Application Ready
       ↓
Git Repository
       ↓
GitHub Repository
       ↓
GitHub Actions
       ↓
Build Application
       ↓
GitHub OIDC Authentication
       ↓
AWS STS
       ↓
IAM Role
       ↓
Temporary AWS Credentials
       ↓
Amazon S3
       ↓
Amazon CloudFront
       ↓
Live Website
```

The important authentication flow is:

```text
GitHub Actions
       ↓
Request OIDC Token
       ↓
AWS STS
       ↓
Check IAM Trust Policy
       ↓
Assume IAM Role
       ↓
Temporary AWS Credentials
       ↓
Use IAM Permissions
       ↓
Access AWS Services
```

---

# 3. Prerequisites

Before starting, you should have:

### GitHub

- GitHub account
- GitHub repository
- GitHub Actions enabled

### AWS

- AWS account
- IAM access
- Amazon S3
- Amazon CloudFront

### Application

The application should have a working production build.

Example:

```bash
npm run build
```

The build should generate something similar to:

```text
dist/
├── index.html
└── assets/
```

---

# 4. Example Project Values

The examples in this documentation use placeholders.

Replace them with your own values when implementing the setup.

```text
YOUR_GITHUB_USERNAME
YOUR_REPOSITORY
YOUR_AWS_ACCOUNT_ID
YOUR_AWS_REGION
YOUR_S3_BUCKET_NAME
YOUR_CLOUDFRONT_DISTRIBUTION_ID
YOUR_IAM_ROLE_NAME
```

For example:

```text
YOUR_GITHUB_USERNAME = your-github-username
YOUR_REPOSITORY = your-repository
YOUR_AWS_ACCOUNT_ID = 123456789012
YOUR_S3_BUCKET_NAME = your-website-bucket
YOUR_CLOUDFRONT_DISTRIBUTION_ID = EXAMPLE123456
YOUR_IAM_ROLE_NAME = GitHubActionsDeploymentRole
```

> **Security note:** Never publish AWS access keys, secret keys, GitHub tokens, OIDC tokens, passwords, or other credentials in a public repository.

---

# 5. Step 1 — Push the Application to GitHub

The completed application is first stored in a GitHub repository.

Flow:

```text
Local Application
       ↓
Git
       ↓
GitHub Repository
```

Example:

```text
YOUR_GITHUB_USERNAME/YOUR_REPOSITORY
```

The main deployment branch can be:

```text
main
```

### Purpose

GitHub stores the source code and provides the source code to GitHub Actions during deployment.

---

# 6. Step 2 — Create the S3 Bucket

Create an Amazon S3 bucket to store the production build files.

Flow:

```text
dist/
  ↓
S3 Bucket
```

Recommended configuration:

```text
Block Public Access = ON

Object Ownership = Bucket owner enforced

Static Website Hosting = OFF
```

The S3 bucket remains private.

### Why keep S3 private?

Users should not access the S3 bucket directly.

The intended architecture is:

```text
User
  ↓
CloudFront
  ↓
Private S3
```

CloudFront is responsible for delivering the website to users.

---

# 7. Step 3 — Create the CloudFront Distribution

Create an Amazon CloudFront distribution using the S3 bucket as the origin.

Flow:

```text
User
  ↓
CloudFront
  ↓
S3
```

Important settings:

```text
Origin = S3 bucket

Default Root Object = index.html

Viewer Protocol Policy = Redirect HTTP to HTTPS
```

### Purpose

CloudFront delivers the application to users using HTTPS and CDN caching.

---

# 8. Step 4 — Configure CloudFront to Access Private S3

Because the S3 bucket is private, CloudFront needs permission to read the files.

Create and configure:

```text
Origin Access Control (OAC)
```

Flow:

```text
CloudFront
    ↓
   OAC
    ↓
Private S3
```

The S3 bucket policy should allow the CloudFront distribution to read objects.

## S3 Bucket Policy

Replace:

```text
YOUR_AWS_ACCOUNT_ID
YOUR_S3_BUCKET_NAME
YOUR_CLOUDFRONT_DISTRIBUTION_ID
```

with your actual values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontRead",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_S3_BUCKET_NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR_AWS_ACCOUNT_ID:distribution/YOUR_CLOUDFRONT_DISTRIBUTION_ID"
        }
      }
    }
  ]
}
```

### Meaning

This policy allows the specified CloudFront distribution to read objects from the private S3 bucket.

---

# 9. Step 5 — Create the GitHub OIDC Provider in AWS

GitHub Actions needs a secure way to authenticate with AWS.

Instead of storing permanent AWS access keys in GitHub, we can use **OpenID Connect (OIDC)**.

Create an OIDC provider in:

```text
AWS Console
    ↓
IAM
    ↓
Identity providers
    ↓
Add provider
```

Use:

```text
Provider URL:

https://token.actions.githubusercontent.com
```

Audience:

```text
sts.amazonaws.com
```

---

## What do Provider and Audience mean?

### Provider

The provider answers:

```text
Who issued the identity token?
```

Answer:

```text
GitHub
```

### Audience

The audience answers:

```text
Who is this token intended for?
```

Answer:

```text
AWS STS
```

So:

```text
Provider
   ↓
GitHub

Audience
   ↓
AWS STS
```

---

# 10. Step 6 — Create the IAM Deployment Role

Create an IAM role for GitHub Actions.

Example:

```text
YOUR_IAM_ROLE_NAME
```

For example:

```text
GitHubActionsDeploymentRole
```

The role is used by GitHub Actions to temporarily access AWS.

The basic flow is:

```text
GitHub Actions
      ↓
OIDC Token
      ↓
AWS STS
      ↓
IAM Role
```

An IAM role has two important policy concepts:

```text
1. Trust Policy
2. Permissions Policy
```

---

# 11. Step 7 — Configure the IAM Trust Policy

The trust policy answers:

> Who is allowed to assume this IAM role?

The GitHub Actions workflow may use a GitHub environment such as:

```yaml
environment: production
```

If the workflow uses the `production` environment, the OIDC subject can identify the repository and environment.

## Trust Policy

Replace:

```text
YOUR_AWS_ACCOUNT_ID
YOUR_GITHUB_USERNAME
YOUR_REPOSITORY
```

with your values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/YOUR_REPOSITORY:environment:production"
        }
      }
    }
  ]
}
```

---

## Understanding the Trust Policy

The important part is:

```text
repo:YOUR_GITHUB_USERNAME/YOUR_REPOSITORY:environment:production
```

Conceptually:

```text
GitHub
   ↓
Repository
   ↓
production environment
   ↓
Allowed to assume the IAM role
```

The trust policy does **not** simply say:

> "Anyone from GitHub can use this role."

It restricts which GitHub identity is trusted.

The conditions can check claims such as:

```text
Issuer / Provider
Audience
Subject
Repository
Environment
```

---

# 12. Step 8 — Configure IAM Permissions

After GitHub is allowed to assume the role, the role needs permission to perform the deployment.

For S3:

```text
s3:ListBucket
s3:PutObject
s3:DeleteObject
```

For CloudFront:

```text
cloudfront:CreateInvalidation
```

## IAM Permissions Policy

Replace:

```text
YOUR_AWS_ACCOUNT_ID
YOUR_S3_BUCKET_NAME
YOUR_CLOUDFRONT_DISTRIBUTION_ID
```

with your values.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::YOUR_S3_BUCKET_NAME"
    },
    {
      "Sid": "ManageWebsiteObjects",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::YOUR_S3_BUCKET_NAME/*"
    },
    {
      "Sid": "CloudFrontInvalidation",
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation"
      ],
      "Resource": "arn:aws:cloudfront::YOUR_AWS_ACCOUNT_ID:distribution/YOUR_CLOUDFRONT_DISTRIBUTION_ID"
    }
  ]
}
```

---

## Trust Policy vs Permissions Policy

This is one of the most important concepts.

```text
Trust Policy
     ↓
WHO can assume the role?
```

```text
Permissions Policy
     ↓
WHAT can the role do?
```

A simple way to remember it:

> **Trust Policy = Who are you?**  
> **Permissions Policy = What are you allowed to do?**

---

# 13. Step 9 — Create the GitHub Production Environment

In GitHub:

```text
Repository
    ↓
Settings
    ↓
Environments
    ↓
Create environment
```

Create:

```text
production
```

The GitHub Actions job can then use:

```yaml
environment: production
```

### Purpose

The environment identifies the deployment as a production deployment.

It can also become part of the OIDC subject used by the IAM trust policy.

---

# 14. Step 10 — Configure GitHub Variables

The workflow needs configuration values such as:

```text
AWS_REGION
S3_BUCKET
CLOUDFRONT_DISTRIBUTION_ID
AWS_ROLE_ARN
```

Example:

```text
AWS_REGION = YOUR_AWS_REGION

S3_BUCKET = YOUR_S3_BUCKET_NAME

CLOUDFRONT_DISTRIBUTION_ID = YOUR_CLOUDFRONT_DISTRIBUTION_ID

AWS_ROLE_ARN = arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/YOUR_IAM_ROLE_NAME
```

For example:

```text
AWS_ROLE_ARN =
arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/GitHubActionsDeploymentRole
```

These are configuration values, not credentials.

> Do not put AWS access keys or secret keys into the repository.

---

# 15. Step 11 — Create the GitHub Actions Workflow

The workflow automates the deployment.

The workflow performs these steps:

```text
Checkout Code
      ↓
Install Dependencies
      ↓
Build Application
      ↓
Verify Build
      ↓
Authenticate with AWS
      ↓
Upload to S3
      ↓
Invalidate CloudFront
      ↓
Deployment Complete
```

## Example Workflow

```yaml
name: Deploy Application to AWS

on:
  push:
    branches:
      - main

  workflow_dispatch:

permissions:
  contents: read
  id-token: write

concurrency:
  group: production-deployment
  cancel-in-progress: true

env:
  AWS_REGION: ${{ vars.AWS_REGION || secrets.AWS_REGION }}
  S3_BUCKET: ${{ vars.S3_BUCKET || secrets.S3_BUCKET }}
  CLOUDFRONT_DISTRIBUTION_ID: ${{ vars.CLOUDFRONT_DISTRIBUTION_ID || secrets.CLOUDFRONT_DISTRIBUTION_ID }}
  AWS_ROLE_ARN: ${{ vars.AWS_ROLE_ARN || secrets.AWS_ROLE_ARN }}

jobs:
  deploy:
    name: Build and Deploy Application
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout source code
        uses: actions/checkout@v5

      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: 24
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Verify build output
        run: |
          if [ ! -f "./dist/index.html" ]; then
            echo "::error::dist/index.html was not generated"
            exit 1
          fi

          echo "Build successful!"
          echo "Contents of dist:"
          ls -lah ./dist

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ env.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: |
          aws sts get-caller-identity

      - name: Deploy website to S3
        run: |
          aws s3 sync ./dist/ "s3://${S3_BUCKET}/" --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id "${CLOUDFRONT_DISTRIBUTION_ID}" \
            --paths "/*"

      - name: Deployment completed
        run: |
          echo "Deployment completed successfully!"
```

---

# 16. Understanding `id-token: write`

One important line in the workflow is:

```yaml
permissions:
  contents: read
  id-token: write
```

The important permission for OIDC is:

```yaml
id-token: write
```

This allows the GitHub Actions job to request an OIDC identity token from GitHub.

It does **not** mean that GitHub receives permanent AWS credentials.

---

# 17. Step 12 — Build the Application

GitHub Actions runs:

```bash
npm ci
npm run build
```

The build creates:

```text
dist/
```

The workflow verifies:

```text
dist/index.html
```

If the file does not exist, deployment stops.

Flow:

```text
Source Code
     ↓
npm ci
     ↓
npm run build
     ↓
dist/
```

---

# 18. Step 13 — Authenticate GitHub Actions with AWS

This is the most important part of the OIDC flow.

The authentication process is:

```text
GitHub Actions
      ↓
Requests OIDC Token
      ↓
AWS STS
      ↓
Checks IAM Trust Policy
      ↓
Assumes IAM Role
      ↓
Temporary AWS Credentials
```

No permanent AWS access key is required for this authentication flow.

---

# 19. How OIDC Authentication Works

GitHub Actions requests an OIDC token from GitHub.

The token contains identity information, commonly represented by claims such as:

```text
iss
aud
sub
```

Conceptually:

```text
iss = Who issued the token?
aud = Who is the token intended for?
sub = Which GitHub identity does this token represent?
```

For example:

```text
iss
 ↓
GitHub OIDC provider

aud
 ↓
AWS STS

sub
 ↓
Repository + Environment
```

---

# 20. AWS STS Receives the OIDC Token

AWS Security Token Service (STS) receives the identity token.

The workflow uses the AWS action:

```yaml
uses: aws-actions/configure-aws-credentials@v5
```

with:

```yaml
role-to-assume: ${{ env.AWS_ROLE_ARN }}
```

AWS STS evaluates whether the token satisfies the IAM role's trust policy.

Conceptually:

```text
GitHub OIDC Token
       ↓
AWS STS
       ↓
Trust Policy Check
       ↓
Allowed?
   ↙       ↘
 YES        NO
  ↓          ↓
Temporary   Access
Credentials Denied
```

---

# 21. Temporary AWS Credentials

If the trust policy conditions are satisfied, AWS STS provides temporary AWS credentials.

The important concept is:

```text
GitHub does not need a permanent AWS access key.
```

Instead:

```text
OIDC Token
    ↓
AWS STS
    ↓
Temporary Credentials
```

Those temporary credentials are then used by the GitHub Actions job to perform the AWS operations allowed by the IAM permissions policy.

---

# 22. Trust Policy and Permissions Policy Together

The complete idea is:

```text
OIDC Token
     ↓
AWS STS
     ↓
Trust Policy
     ↓
"Is this identity allowed to assume this role?"
     ↓
YES
     ↓
Temporary AWS Credentials
     ↓
Permissions Policy
     ↓
"What can this role do?"
     ↓
S3 / CloudFront
```

This distinction is extremely important.

### Trust Policy

Answers:

```text
WHO can assume the role?
```

### Permissions Policy

Answers:

```text
WHAT can the role do?
```

---

# 23. Step 14 — Upload the Build to S3

After AWS authentication succeeds:

```text
dist/
  ↓
S3 Bucket
```

The workflow runs:

```bash
aws s3 sync ./dist/ "s3://${S3_BUCKET}/" --delete
```

### Purpose

This copies the latest production build to S3.

The:

```text
--delete
```

option removes S3 files that are no longer present in the new `dist/` folder.

---

# 24. Step 15 — Invalidate CloudFront

CloudFront may have older files cached.

The workflow therefore creates an invalidation:

```bash
aws cloudfront create-invalidation \
  --distribution-id "${CLOUDFRONT_DISTRIBUTION_ID}" \
  --paths "/*"
```

Flow:

```text
New Build
    ↓
S3 Updated
    ↓
CloudFront Cache Invalidation
    ↓
CloudFront Gets Updated Files
```

---

# 25. Step 16 — Application Is Live

The final user request flow is:

```text
User
  ↓
CloudFront
  ↓
Private S3
  ↓
Latest Application
```

The deployment is complete.

---

# 26. Complete Architecture

```text
                         GitHub
                            │
                            │ git push
                            ▼
                    GitHub Actions
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
        Build Application           OIDC Authentication
              │                           │
              ▼                           ▼
            dist/                     OIDC Token
                                          │
                                          ▼
                                      AWS STS
                                          │
                                          ▼
                                   IAM Trust Policy
                                          │
                                          ▼
                              Temporary AWS Credentials
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                              ▼                       ▼
                             S3                  CloudFront
                              │                       │
                              └───────────┬───────────┘
                                          │
                                          ▼
                                     Live Website
```

---

# 27. Complete OIDC Authentication Flow

```text
Developer
    ↓
git push
    ↓
GitHub
    ↓
GitHub Actions
    ↓
GitHub OIDC
    ↓
OIDC Token
    ↓
AWS STS
    ↓
IAM Trust Policy
    ↓
IAM Role
    ↓
Temporary AWS Credentials
    ↓
IAM Permissions Policy
    ↓
AWS Services
```

---

# 28. Important Components

| Component | Purpose |
|---|---|
| Git | Tracks application changes |
| GitHub | Stores source code |
| GitHub Actions | Automates the deployment |
| OIDC | Allows GitHub to identify itself to AWS |
| AWS STS | Exchanges the trusted OIDC identity for temporary AWS credentials |
| IAM | Controls AWS access |
| IAM Trust Policy | Decides who can assume the role |
| IAM Permissions Policy | Decides what the role can do |
| S3 | Stores the built application |
| CloudFront | Delivers the application to users |
| OAC | Allows CloudFront to securely access private S3 |
| Invalidation | Removes old CloudFront cached content |

---

# 29. Why Use OIDC?

Without OIDC, a common approach would be to create long-lived AWS access keys and store them in GitHub.

Conceptually:

```text
GitHub Actions
      ↓
Permanent AWS Access Key
      ↓
AWS
```

With OIDC:

```text
GitHub Actions
      ↓
OIDC Token
      ↓
AWS STS
      ↓
Temporary Credentials
      ↓
AWS
```

The OIDC approach avoids storing a long-lived AWS access key for this authentication flow.

---

# 30. Key Concept to Remember

The easiest way to remember the entire process is:

```text
OIDC proves the GitHub identity
            ↓
Trust Policy decides whether that identity is trusted
            ↓
STS provides temporary AWS credentials
            ↓
Permissions Policy decides what those credentials can do
```

Or even shorter:

> **OIDC = Identity**  
> **Trust Policy = Who can enter**  
> **STS = Temporary credentials**  
> **Permissions Policy = What they can do**

---

# 31. Deployment Checklist

## Application

- [ ] Application is completed
- [ ] `npm run build` works
- [ ] `dist/index.html` exists
- [ ] Code is pushed to GitHub

## S3

- [ ] S3 bucket created
- [ ] Block Public Access enabled
- [ ] Object Ownership = Bucket owner enforced
- [ ] Static Website Hosting disabled

## CloudFront

- [ ] CloudFront distribution created
- [ ] S3 bucket configured as origin
- [ ] OAC configured
- [ ] S3 bucket policy configured
- [ ] Default Root Object = `index.html`
- [ ] HTTP redirects to HTTPS

## IAM / OIDC

- [ ] GitHub OIDC provider created
- [ ] Provider URL configured
- [ ] Audience = `sts.amazonaws.com`
- [ ] IAM deployment role created
- [ ] Trust policy configured
- [ ] Permissions policy configured

## GitHub

- [ ] `production` environment created
- [ ] `AWS_REGION` configured
- [ ] `S3_BUCKET` configured
- [ ] `CLOUDFRONT_DISTRIBUTION_ID` configured
- [ ] `AWS_ROLE_ARN` configured
- [ ] GitHub Actions workflow added
- [ ] `id-token: write` permission configured

## Deployment

- [ ] GitHub Actions starts
- [ ] Dependencies install successfully
- [ ] Application builds successfully
- [ ] OIDC authentication succeeds
- [ ] AWS STS provides temporary credentials
- [ ] S3 upload succeeds
- [ ] CloudFront invalidation succeeds
- [ ] Website is accessible through CloudFront

---

# 32. Public Repository Security Checklist

Before publishing this documentation publicly, make sure you have removed:

- AWS Access Key IDs
- AWS Secret Access Keys
- GitHub Personal Access Tokens
- OIDC tokens
- Passwords
- API keys
- Private keys
- Database credentials
- Secret environment variables
- Private URLs containing credentials
- Personal email addresses if they should remain private

The following are examples of values that should be replaced with placeholders:

```text
YOUR_GITHUB_USERNAME
YOUR_REPOSITORY
YOUR_AWS_ACCOUNT_ID
YOUR_AWS_REGION
YOUR_S3_BUCKET_NAME
YOUR_CLOUDFRONT_DISTRIBUTION_ID
YOUR_IAM_ROLE_NAME
```

---

# 33. Final Flow to Remember

```text
CODE
  ↓
GITHUB
  ↓
GITHUB ACTIONS
  ↓
BUILD
  ↓
OIDC
  ↓
AWS STS
  ↓
IAM TRUST POLICY
  ↓
TEMPORARY CREDENTIALS
  ↓
IAM PERMISSIONS
  ↓
S3
  ↓
CLOUDFRONT
  ↓
USER
```

### In one sentence:

**GitHub stores the code, GitHub Actions performs the deployment, OIDC identifies GitHub to AWS, STS provides temporary credentials, IAM controls what can be accessed, S3 stores the build, and CloudFront delivers the application.**
