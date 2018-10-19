# archives-rr-blueprints
The "Infrastructure as Code" repo for all pieces. Will contain Cloud Formation Templates, Ansible playbooks, deploy scripts, etc for all components of the new system.

Note: It is highly recommended you use something like https://github.com/awslabs/git-secrets to prevent pushing AWS secrets to the repo

# Requirements
Before you begin, check that you have the following:
  - A role with permissions to deploy cloudformations. In most cases, will require permissions to create IAM roles/policies (see [Permissions Required to Access IAM Resources](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_permissions-required.html))
  - Ability to manage DNS for your organization to validate certificates (see [Use DNS to Validate Domain Ownership](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-dns.html))
  - A policy that allows your approvers to approve pipelines (see [Grant Approval Permissions to an IAM User in AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/approvals-iam-permissions.html))
  - Must have awscli installed if using the example deploy commands

# Deploy
TODO:
* [ ] Add stack diagram. Important to note the Network and App-Infrastructure stacks are intended to be shared per env. Ex: Only one of each of these exist in dev, but you can have multiple dev instances of service/webcomponent stacks for each developer.
* [ ] Explain why we have the separation we do, reference [Organize Your Stacks By Lifecycle and Ownership](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html#organizingstacks)

## Deploy Shared Infrastructure
Before you can deploy any of the other stacks, you must deploy some prerequisite pieces of shared infrastructure. These are required by both the application components and the CI/CD stacks that test and deploy those application components.

TODO: Add example of exporting an existing network

### Infrastructure stack
Note: This will require adding a DNS entry to validate the certificate created by the stack. The stack will not complete until this is done. See https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-dns.html.

```console
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --template-file deploy/cloudformation/app-infrastructure.yml \
  --stack-name archives-rr-app-infrastructure \
  --tags ProjectName=archives-rr \
  --parameter-overrides NameTag='testaccount-archives-rrappinfrastructure-dev' ContactTag='me@myhost.com' OwnerTag='me'
```

## Deploy Application Components

### IIIF Image Viewer Webcomponent stack
```console
aws cloudformation deploy \
  --stack-name archives-rr-image-webcomponent-dev \
  --template-file deploy/cloudformation/static-host.yml \
  --tags ProjectName=archives-rr \
  --parameter-overrides NameTag='testaccount-archives-rrimagewebcomponent-dev' ContactTag='me@myhost.com' OwnerTag='myid'
```

### Main Website stack
```console
aws cloudformation deploy \
  --stack-name archives-rr-website-dev \
  --template-file deploy/cloudformation/static-host.yml \
  --tags ProjectName=archives-rr \
  --parameter-overrides NameTag='testaccount-archives-rrwebsite-dev' ContactTag='me@myhost.com' OwnerTag='myid'
```

## Deploy CI/CD
Before you begin see https://developer.github.com/v3/auth/#via-oauth-tokens for how to generate an OAuth token for use with these pipelines.

### IIIF Image Viewer Pipeline
This will deploy to test, then to production, so it expects two different image-viewer stacks to exist, ex: "archives-rr-image-webcomponent-test" and "archives-rr-image-webcomponent-prod".

```console
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --stack-name archives-rr-image-webcomponent-pipeline \
  --template-file deploy/cloudformation/static-host-pipeline.yml \
  --tags ProjectName=archives-rr \
  --parameter-overrides OAuth=my_oauth_key Approvers=me@myhost.com \
    SourceRepoOwner=ndlib SourceRepoName=image-viewer BuildScriptsDir='build' BuildOutputDir='dist' \
    TestStackName=archives-rr-image-webcomponent-test ProdStackName=archives-rr-image-webcomponent-prod \
    NameTag='testaccount-archives-rrimagewebcomponentpipeline' ContactTag='me@myhost.com' OwnerTag='myid'
```

### Website Pipeline
This will deploy to test, then to production, so it expects two different website stacks to exist, ex: "archives-rr-website-test" and "archives-rr-website-prod".

```console
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --stack-name archives-rr-website-pipeline \
  --template-file deploy/cloudformation/static-host-pipeline.yml \
  --tags ProjectName=archives-rr \
  --parameter-overrides OAuth=my_oauth_key Approvers=me@myhost.com \
    SourceRepoOwner=ndlib SourceRepoName=archives-rr-website \
    TestStackName=archives-rr-website-test ProdStackName=archives-rr-website-prod \
    NameTag='testaccount-archives-rrwebsitepipeline' ContactTag='me@myhost.com' OwnerTag='myid'
```

#### Approval message
Once the pipeline reaches the UAT step, it will send an email to the approvers list and wait until it's either approved or rejected. Here's an example of the message.

```email
Approve or reject: https://console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID/Approval/ManualApprovalOfTestEnvironment/approve/approval-id
Additional information: You can review these changes at https://testurl. Once approved, this will be deployed to https://produrl.
Deadline: This review request will expire on 2018-10-15T20:36Z
```

The link given in the email will take the user directly to the approval modal:

* [ ] Add screenshot of approval here

Note: The user must be logged in and have the appropriate permissions to approve pipelines.

### Pipeline Monitoring
Use this stack if you want to notify an email address of pipeline events. It is currently written to only accept a single email address, so it's recommended you use a mailing list for the Receivers parameter.

Here's an example of adding monitoring to the image-webcomponent-pipeline
```console
aws cloudformation deploy \
  --stack-name archives-rr-image-webcomponent-pipeline-monitoring \
  --template-file deploy/cloudformation/pipeline-monitoring.yml \
  --tags ProjectName=archives-rr \
  --parameter-overrides PipelineStackName=archives-rr-image-webcomponent-pipeline Receivers=me@myhost.com
```

Here's an example of adding monitoring to the website-pipeline
```console
aws cloudformation deploy \
  --stack-name archives-rr-website-pipeline-monitoring \
  --template-file deploy/cloudformation/pipeline-monitoring.yml \
  --tags ProjectName=archives-rr \
  --parameter-overrides PipelineStackName=archives-rr-website-pipeline Receivers=me@myhost.com
```

Here's an example of adding monitoring to the image-service-pipeline
```console
aws cloudformation deploy \
  --stack-name archives-rr-image-service-pipeline-monitoring \
  --template-file deploy/cloudformation/pipeline-monitoring.yml \
  --tags ProjectName=archives-rr \
  --parameter-overrides PipelineStackName=archives-rr-image-service-pipeline Receivers=me@myhost.com
```

#### Examples of the notifications:
##### Started
The pipeline archives-rr-image-webcomponent-pipeline has started. To view the pipeline, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.

##### Success
The pipeline archives-rr-image-webcomponent-pipeline has successfully deployed to production. To view the pipeline, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.

##### Source failure
Failed to pull the source code for archives-rr-image-webcomponent-pipeline. To view the current execution, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.

##### Build failure
Failed to build archives-rr-image-webcomponent-pipeline. To view the pipeline, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.

##### Deploy to test failure
Build for archives-rr-image-webcomponent-pipeline failed to deploy to test stack. To view the pipeline, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.

##### Approval failure
Build for archives-rr-image-webcomponent-pipeline was rejected either due to a QA failure or UAT rejection. To view the pipeline, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.

##### Deploy to production failure
Build for archives-rr-image-webcomponent-pipeline failed to deploy to production. To view the pipeline, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.

##### Generic resume after a failure
The pipeline archives-rr-image-webcomponent-pipeline has changed state to RESUMED. To view the pipeline, go to https://us-west-2.console.aws.amazon.com/codepipeline/home?region=us-west-2#/view/archives-rr-image-webcomponent-pipeline-CodePipeline-ID.
