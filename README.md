# archives-rr-blueprints
The "Infrastructure as Code" repo for all pieces. Will contain Cloud Formation Templates, Ansible playbooks, deploy scripts, etc for all components of the archives-rr system.

Note: It is highly recommended you use something like https://github.com/awslabs/git-secrets to prevent pushing AWS secrets to the repo

# Requirements
Before you begin, check that you have the following:
  - A role with permissions to deploy cloudformations. In most cases, will require permissions to create IAM roles/policies (see [Permissions Required to Access IAM Resources](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_permissions-required.html))
  - Ability to manage DNS for your organization to validate certificates (see [Use DNS to Validate Domain Ownership](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-dns.html))
  - A policy that allows your approvers to approve pipelines (see [Grant Approval Permissions to an IAM User in AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/approvals-iam-permissions.html))
  - Must have awscli installed if using the example deploy commands

# Deploy
## Infrastructure stack
```console
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --template-file deploy/cloudformation/app-infrastructure.yml \
  --stack-name archives-rr-app-infrastructure \
  --tags ProjectName=archives-rr Name='accountname-archives-rr-appinfrastructure-env' Contact='me@myhost.com' Owner='myid'
```
## Website test stack - FQDN Unknown
```console
aws cloudformation deploy \
  --stack-name archives-rr-website-test \
  --template-file deploy/cloudformation/static-host.yml \
  --tags ProjectName=archives-rr Name='accountname-archives-rr-website-prep' Contact='me@myhost.com' Owner='myid'
```
## Website test stack with a Known DNS Entry
```console
aws cloudformation deploy \
  --stack-name archives-rr-website-test \
  --template-file deploy/cloudformation/static-host.yml \
  --tags ProjectName=archives-rr Name='accountname-archives-rr-website-prep' Contact='me@myhost.com' Owner='myid' \
  --parameter-overrides FQDN='fqdn-of-test-service'
```  
## Website prod stack - FQDN Unknown
```console
aws cloudformation deploy \
  --stack-name archives-rr-website-prod \
  --template-file deploy/cloudformation/static-host.yml \
  --tags ProjectName=archives-rr Name='accountname-archives-rr-website-prod' Contact='me@myhost.com' Owner='myid'
```
## Website prod stack with a Known DNS Entry
```console
aws cloudformation deploy \
  --stack-name archives-rr-website-prod \
  --template-file deploy/cloudformation/static-host.yml \
  --tags ProjectName=archives-rr Name='accountname-archives-rr-website-prod' Contact='me@myhost.com' Owner='myid' \
  --parameter-overrides FQDN='fqdn-of-prod-service'
```
## Website Pipeline
Before you begin see https://developer.github.com/v3/auth/#via-oauth-tokens for how to generate an OAuth token for use with these pipelines. This will deploy to test, then to production, so it expects two different website stacks to exist, ex: "archives-rr-website-test" and "archives-rr-website-prod".

```console
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --stack-name archives-rr-website-pipeline \
  --template-file deploy/cloudformation/static-host-pipeline.yml \
  --tags ProjectName=archives-rr Name='accountname-archives-rr-websitepipeline-prod' Contact='me@myhost.com' Owner='myid' \
  --parameter-overrides OAuth=my_oauth_key Approvers=me@myhost.com \
    SourceRepoOwner=ndlib SourceRepoName=archives-rr \
    TestStackName=archives-rr-website-test ProdStackName=archives-rr-website-prod
```

## Pipeline Monitoring
Use this stack if you want to notify an email address of pipeline events. It is currently written to only accept a single email address, so it's recommended you use a mailing list for the Receivers parameter.

```console
aws cloudformation deploy \
  --stack-name archives-rr-website-pipeline-monitoring \
  --template-file deploy/cloudformation/pipeline-monitoring.yml \
  --tags ProjectName=archives-rr Name='accountname-archives-rr-websitepipeline-prod' Contact='me@myhost.com' Owner='myid' \
  --parameter-overrides PipelineStackName=archives-rr-website-pipeline Receivers=me@myhost.com
```
