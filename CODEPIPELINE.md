# GitHub → CodePipeline → CodeBuild → S3 → CloudFront

## 1. Objective

Extend the existing CI/CD deployment by introducing **AWS CodePipeline** as the orchestration service.

The final architecture is:

```text
GitHub
   ↓
CodePipeline
   ↓
CodeBuild
   ↓
S3
   ↓
CloudFront

CodePipeline is responsible for orchestrating the workflow, CodeBuild performs the build, S3 stores the website, and CloudFront delivers the website to users.

2. Existing Infrastructure

The following resources were already created:

GitHub Repository:
cicd-testing-dva-co2

S3 Bucket:
cicd-testing-dva-co2

CloudFront Distribution:
E2UFMMUXW0AFY4

CloudFront OAC:
cicd-testing-dva-co2-oac

CodeBuild Project:
cicd-testing-dva-co2-build

GitHub Connection:
cicd-testing-dva-co2-github

The S3 bucket is private.

Static website hosting is disabled.

CloudFront accesses the S3 bucket using Origin Access Control (OAC).

3. Why Add CodePipeline?

Initially, the deployment flow was:

GitHub
   ↓
GitHub Webhook
   ↓
CodeBuild
   ↓
S3
   ↓
CloudFront

CodeBuild was responsible for both building and deploying the website.

After introducing CodePipeline, we wanted to separate those responsibilities:

GitHub
   ↓
CodePipeline
   ↓
CodeBuild
   ↓
S3
   ↓
CloudFront

The responsibilities are now:

Service	Responsibility
GitHub	Source code
CodeConnections	Secure GitHub connection
CodePipeline	Orchestrates CI/CD
CodeBuild	Executes build commands
S3	Stores deployed website
CloudFront	Delivers website
OAC	Allows CloudFront to access private S3
4. Create the CodePipeline

Open:

AWS Console → CodePipeline → Create pipeline

Choose:

Category:
Build custom pipeline

Do not use the pre-built templates such as:

Push to ECR
Deploy to ECS Fargate
Deploy to CloudFormation
Terraform Deploy to AWS

A custom pipeline allows the individual stages to be configured explicitly.

5. Pipeline Settings

Use:

Pipeline name:
cicd-testing-dva-co2-pipeline
Execution mode

Select:

Superseded

Execution modes determine what happens when a new pipeline execution starts while another execution is running.

Superseded

A newer execution takes priority over an older in-progress execution.

Example:

Execution 1 → Running
Execution 2 → Starts
Execution 1 → Superseded
Execution 2 → Continues

This is useful for deployments where the latest source version is more important than deploying every intermediate version.

Queued

New executions wait for previous executions to finish.

Execution 1 → Running
Execution 2 → Waiting
Execution 3 → Waiting
Parallel

Multiple executions can run simultaneously.

Execution 1 → Running
Execution 2 → Running
Execution 3 → Running

For this website deployment, Superseded was selected.

Service role

Choose:

New service role

Allow AWS CodePipeline to create the service role.

This allows AWS to create the IAM role required by CodePipeline.

6. Configure the Source Stage

Select:

Source provider:
GitHub (via GitHub App)

Use the existing connection:

cicd-testing-dva-co2-github

Repository:

cicd-testing-dva-co2

Branch:

main
Output artifact format

Select:

CodePipeline default

This creates the standard CodePipeline ZIP artifact.

We do not need the Full clone option because CodeBuild only needs the source files for this static website.

The flow becomes:

GitHub
   ↓
SourceArtifact
   ↓
CodeBuild
7. Configure the Pipeline Webhook

Enable:

Start your pipeline on push and pull request events

For this lab, we only want pushes to the main branch to trigger deployments.

Configure:

Event type:
Push

Start pipeline under these conditions:

Filter type:
Branch

Branches or patterns:
main

Leave:

Don't start pipeline under these conditions

empty.

The result is:

git push origin main
        ↓
GitHub PUSH event
        ↓
Branch = main?
        ↓
YES
        ↓
Start CodePipeline

A push to another branch does not start the deployment pipeline.

8. Configure the Build Stage

Select:

Build provider:
AWS CodeBuild

Use the existing project:

cicd-testing-dva-co2-build

Input artifact:

SourceArtifact

Do not use the newer managed Commands build option for this lab.

We already created a CodeBuild project and want to understand the standard:

CodePipeline → CodeBuild

integration.

The flow is:

GitHub
   ↓
SourceArtifact
   ↓
CodeBuild
9. Configure the Test Stage

For this simple static website, no automated test stage is required.

Leave the Test stage empty.

Test:
Skipped

A test stage can be introduced later when automated tests are available.

10. Configure the Deploy Stage

Select:

Deploy provider:
Amazon S3

Region:

eu-west-1

Input artifact:

BuildArtifact

Bucket:

cicd-testing-dva-co2

S3 object key:

Leave empty

Enable:

Extract file before deploy

This is important because CodePipeline produces a ZIP artifact.

We want the contents extracted into the S3 bucket:

S3
├── index.html
└── ...

rather than storing the deployment as a ZIP file.

Leave automatic rollback and automatic retry disabled for this basic lab.

11. Change the CodeBuild Responsibility

Before CodePipeline, CodeBuild was deploying directly to S3.

The old buildspec.yml contained:

version: 0.2

phases:
  build:
    commands:
      - echo "Deploying website to S3..."
      - aws s3 sync . s3://cicd-testing-dva-co2

artifacts:
  files:
    - index.html

This meant CodeBuild was responsible for deployment.

After introducing CodePipeline, this responsibility should move to the CodePipeline S3 Deploy stage.

The new buildspec.yml is:

version: 0.2

phases:
  build:
    commands:
      - echo "Building website..."
      - echo "Build completed successfully"

artifacts:
  files:
    - index.html

Now CodeBuild produces the artifact but does not deploy it.

The responsibilities become:

CodeBuild:
Build → BuildArtifact

CodePipeline:
BuildArtifact → S3
12. Commit the Updated Buildspec

Commit the new buildspec:

git add buildspec.yml
git commit -m "Move deployment to CodePipeline"
git push origin main

The source repository should contain:

cicd-testing-dva-co2/
├── index.html
└── buildspec.yml
13. Create the Pipeline

Review the configuration:

Pipeline:
cicd-testing-dva-co2-pipeline

Source:
GitHub
cicd-testing-dva-co2
main

Build:
AWS CodeBuild
cicd-testing-dva-co2-build

Test:
None

Deploy:
Amazon S3
cicd-testing-dva-co2

Create the pipeline.

14. Successful Pipeline Execution

A successful execution produced:

Source     → Succeeded
Build      → Succeeded
Deploy     → Succeeded

The complete flow was:

GitHub
   ↓
Source
   ↓
CodeBuild
   ↓
BuildArtifact
   ↓
S3 Deploy
   ↓
S3

This confirmed that CodePipeline was successfully orchestrating the deployment.

15. Disable the Old CodeBuild Webhook

Before CodePipeline was introduced, CodeBuild had its own GitHub webhook enabled:

GitHub
   ↓
CodeBuild

Once CodePipeline became responsible for triggering the deployment, this webhook was no longer required.

If it remained enabled, a single GitHub push could trigger:

GitHub
   ├──→ CodeBuild directly
   │
   └──→ CodePipeline
          ↓
        CodeBuild

This creates two independent paths to CodeBuild.

Disable it

Go to:

CodeBuild → Build projects → cicd-testing-dva-co2-build → Edit

Find:

Primary source webhook events

Disable:

Rebuild every time a code change is pushed to this repository

Do not change:

Source provider
GitHub repository
Branch
Connection

Only disable the CodeBuild webhook.

Save/update the project.

The final trigger becomes:

GitHub
   ↓
CodePipeline
   ↓
CodeBuild

CodePipeline is now the single entry point for deployments.

16. Test a New Deployment

Modify index.html.

For example:

<h2>CodePipeline deployment test v2</h2>

Commit and push:

git add .
git commit -m "Update website"
git push origin main

The expected flow is:

GitHub PUSH
     ↓
CodePipeline
     ↓
Source ✓
     ↓
Build ✓
     ↓
Deploy ✓
     ↓
S3

No direct CodeBuild webhook is required.

17. CloudFront Cache Issue

The pipeline successfully deployed the updated index.html to S3, but the CloudFront website initially displayed the previous version.

The important diagnostic step was checking S3 first.

The new HTML was present in S3.

Therefore:

GitHub       ✓
CodePipeline ✓
CodeBuild    ✓
S3           ✓
CloudFront   → Cached old version

The deployment was successful; CloudFront was serving a cached object.

18. CloudFront Invalidation

Create a CloudFront invalidation:

/*

This forces CloudFront to retrieve the updated objects from the origin.

After the invalidation completed, the updated website was displayed successfully.

The complete flow became:

GitHub
   ↓
CodePipeline
   ↓
CodeBuild
   ↓
S3
   ↓
CloudFront
   ↓
Cache Invalidation
   ↓
Updated Website
19. Final Architecture
                    git push
                       │
                       ▼
                ┌─────────────┐
                │   GitHub    │
                └──────┬──────┘
                       │
                       │ GitHub App
                       ▼
              ┌──────────────────┐
              │   CodePipeline   │
              └────────┬─────────┘
                       │
                  Source stage
                       │
                  SourceArtifact
                       │
                       ▼
              ┌──────────────────┐
              │    CodeBuild     │
              │                  │
              │  buildspec.yml   │
              └────────┬─────────┘
                       │
                  BuildArtifact
                       │
                       ▼
              ┌──────────────────┐
              │       S3         │
              │     PRIVATE      │
              └────────┬─────────┘
                       │
                      OAC
                       │
                       ▼
              ┌──────────────────┐
              │   CloudFront     │
              └────────┬─────────┘
                       │
                       ▼
                     User
20. Key Exam Learnings
CodePipeline

CodePipeline is the orchestration service.

It coordinates stages such as:

Source → Build → Test → Deploy
CodeBuild

CodeBuild executes the commands defined in buildspec.yml.

In this architecture, CodeBuild produces the build artifact but does not perform the final S3 deployment.

CodeConnections

CodeConnections provides the authenticated connection between AWS and GitHub.

Webhooks

A webhook allows GitHub events to automatically trigger the pipeline.

For this lab:

PUSH → main → Start CodePipeline
Artifacts

The source stage produces:

SourceArtifact

CodeBuild consumes the source artifact and produces:

BuildArtifact

The S3 deployment stage consumes:

BuildArtifact
CloudFront caching

A successful deployment to S3 does not necessarily mean CloudFront immediately serves the new object.

If an old object is cached, create a CloudFront invalidation such as:

/*
Final CI/CD Flow
Developer
    │
    │ git push origin main
    ▼
GitHub
    │
    │ webhook
    ▼
CodePipeline
    │
    ├── Source
    │
    ├── Build
    │      │
    │      ▼
    │   CodeBuild
    │      │
    │      ▼
    │   BuildArtifact
    │
    └── Deploy
           │
           ▼
          S3
           │
           ▼
       CloudFront
           │
           ▼
         User

The final responsibility model is:

GitHub       = Source
CodePipeline = Orchestration
CodeBuild    = Build
S3           = Deployment target / storage
CloudFront   = Content delivery
OAC          = Secure CloudFront → S3 access