# GitHub → CodeBuild → S3 → CloudFront CI/CD

## 1. Architecture

```text
Developer
   ↓ git push
GitHub
   ↓ Webhook
AWS CodeBuild
   ↓ aws s3 sync
Private S3 Bucket
   ↓ OAC
CloudFront
   ↓
Website
````

**Services:** GitHub, AWS CodeConnections, CodeBuild, IAM, S3 and CloudFront.

> S3 Static Website Hosting is disabled. The S3 bucket remains private and CloudFront accesses it using Origin Access Control (OAC).

---

## 2. Create the GitHub Repository

Create:

```text
cicd-testing-dva-co2
```

Create the local project:

```bash
mkdir cicd-testing-dva-co2
cd cicd-testing-dva-co2
```

Create `index.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>CI/CD Testing</title>
</head>
<body>
    <h1>Hello from AWS!</h1>
    <h2>CI/CD deployment test</h2>
    <p>This website was deployed using GitHub, CodeBuild, S3, and CloudFront.</p>
    <p>Deployment pipeline is working successfully 🚀</p>
</body>
</html>
```

Initialise Git and push:

```bash
git init
git add index.html
git commit -m "Initial website"
git branch -M main
git remote add origin <YOUR_GITHUB_REPO_URL>
git push -u origin main
```

---

## 3. Create the S3 Bucket

Create:

```text
cicd-testing-dva-co2
```

Keep:

```text
Block all public access: ON
Static website hosting: OFF
```

The bucket is intentionally private.

---

## 4. Create CloudFront

Create a CloudFront distribution using the S3 bucket as the origin.

Configure:

```text
Origin path: blank
Origin access: Origin Access Control (OAC)
Signing behavior: Sign requests
Viewer protocol: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Default root object: index.html
```

Create OAC:

```text
cicd-testing-dva-co2-oac
```

Example distribution ID:

```text
E2UFMMUXW0AFY4
```

---

## 5. S3 Bucket Policy for CloudFront

Allow only the CloudFront distribution to read objects:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::cicd-testing-dva-co2/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::922318569961:distribution/E2UFMMUXW0AFY4"
        }
      }
    }
  ]
}
```

Replace the account ID and distribution ID when using different resources.

---

## 6. Connect GitHub to AWS

Go to:

**CodeBuild → Manage account credentials → GitHub**

Use the **AWS-managed GitHub App / AWS Connector for GitHub**.

When installing the connector:

```text
Repository access: Only select repositories
Repository: cicd-testing-dva-co2
```

Create an AWS CodeConnections connection in the **same region as CodeBuild**.

For this lab:

```text
Region: eu-west-1
Connection: cicd-testing-dva-co2
Status: Available
```

---

## 7. CodeConnections Error Encountered

Initially CodeBuild returned:

```text
Webhook creation failed

Access denied to connection arn:aws:codeconnections:...
```

### Cause

The CodeBuild service role did not have permission to use the CodeConnections connection.

### Fix

Add the following to the CodeBuild service role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codeconnections:GetConnectionToken",
        "codeconnections:GetConnection"
      ],
      "Resource": "arn:aws:codeconnections:eu-west-1:922318569961:connection/ab0dcb98-a9a8-44b6-8739-b5b10c9db626"
    }
  ]
}
```

---

## 8. Create CodeBuild Project

Create:

```text
cicd-testing-dva-co2-build
```

### Source

```text
Provider: GitHub
Repository: cicd-testing-dva-co2
Branch: main
Connection: cicd-testing-dva-co2
```

### Environment

```text
Provisioning: On-demand
Image: Managed image
Compute: EC2
Running mode: Container
OS: Amazon Linux
Runtime: Standard
Image: aws/codebuild/amazonlinux-x86_64-standard:6.0
```

Create/select the CodeBuild service role:

```text
cicd-testing-dva-co2-build-service-role
```

---

## 9. Configure the Webhook

Enable:

```text
Rebuild every time a code change is pushed to this repository
```

Use:

```text
Build type: Single build
```

Create one webhook filter group:

```text
Event type: PUSH
Condition: Start a build
Filter: HEAD_REF
Pattern: ^refs/heads/main$
```

This means:

```text
Push to main → Start CodeBuild
```

After saving, verify:

```text
Webhook: Enabled
```

---

## 10. GitHub Webhook Error Encountered

We initially received:

```text
Failed to create webhook.
GitHub API limit reached or permission issue encountered
```

### Cause

The AWS Connector for GitHub had not been properly installed/configured for the repository.

### Fix

Ensure the AWS Connector for GitHub is installed and has access to:

```text
cicd-testing-dva-co2
```

Then ensure the CodeConnections connection is:

```text
Available
```

and selected by the CodeBuild project.

---

## 11. Create `buildspec.yml`

Add this file to the root of the GitHub repository:

```yaml
version: 0.2

phases:
  build:
    commands:
      - echo "Deploying website to S3..."
      - aws s3 sync . s3://cicd-testing-dva-co2 --exclude ".git/*" --exclude "buildspec.yml"

artifacts:
  files:
    - index.html
```

The important command is:

```bash
aws s3 sync . s3://cicd-testing-dva-co2
```

---

## 12. Give CodeBuild S3 Permissions

Create an IAM policy:

```text
cicd-testing-dva-co2-s3-deploy
```

Attach it to:

```text
cicd-testing-dva-co2-build-service-role
```

Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::cicd-testing-dva-co2"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::cicd-testing-dva-co2/*"
    }
  ]
}
```

`ListBucket` uses the bucket ARN; object actions use the `/*` ARN.

---

## 13. Test the Deployment

Commit and push:

```bash
git add .
git commit -m "Configure CI/CD deployment"
git push origin main
```

Do **not** manually start CodeBuild.

The expected flow is:

```text
git push
   ↓
GitHub webhook
   ↓
CodeBuild
   ↓
buildspec.yml
   ↓
aws s3 sync
   ↓
S3
```

Check:

**CodeBuild → Build history**

Expected:

```text
Status: Succeeded
Submitter: GitHub-Hookshot
```

Then check:

**S3 → cicd-testing-dva-co2 → Objects**

You should see:

```text
index.html
```

---

## 14. Test CloudFront

Open:

```text
https://<your-cloudfront-domain>/
```

The website should load through CloudFront.

### CloudFront AccessDenied Error

If CloudFront returns:

```xml
<Error>
    <Code>AccessDenied</Code>
    <Message>Access Denied</Message>
</Error>
```

check:

```text
Default root object: index.html
```

Do not use:

```text
/index.html
```

Also verify:

```text
CloudFront OAC → S3
S3 bucket policy → correct CloudFront distribution ARN
Distribution → fully deployed
```

---

## 15. Final Working Architecture

```text
                 ┌──────────────┐
                 │    GitHub    │
                 └──────┬───────┘
                        │ push
                        ▼
                 ┌──────────────┐
                 │   Webhook    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │  CodeBuild   │
                 │ buildspec.yml│
                 └──────┬───────┘
                        │
                  aws s3 sync
                        │
                        ▼
                 ┌──────────────┐
                 │     S3       │
                 │   PRIVATE    │
                 └──────┬───────┘
                        │ OAC
                        ▼
                 ┌──────────────┐
                 │  CloudFront  │
                 └──────┬───────┘
                        │
                        ▼
                      User
```

## Key Exam Concepts

* **GitHub:** source repository.
* **CodeConnections:** authenticated connection between AWS and GitHub.
* **Webhook:** automatically triggers CodeBuild after a push.
* **CodeBuild:** executes the buildspec and deployment commands.
* **IAM service role:** gives CodeBuild permissions to AWS resources.
* **S3:** stores the website.
* **CloudFront:** delivers the website globally.
* **OAC:** allows CloudFront to securely access a private S3 bucket.
* **CodePipeline:** not used yet. It will be introduced in the next lab to orchestrate the workflow.

### Current workflow

```text
GitHub → Webhook → CodeBuild → S3 → CloudFront
```

### Next workflow

```text
GitHub → CodePipeline → CodeBuild → S3 → CloudFront
```

```
```
