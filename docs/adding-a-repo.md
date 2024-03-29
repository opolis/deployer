Adding a Repository
===================

Repositories can be processed by `opolis/deployer` by configuring a webhook and adding a few files to the repo.

## Configuring the Webhook

"Webhooks" are a means for GitHub to notify third party services that a particular event has occurred on a particular
repository. It is simply a `POST` request to a particular endpoint, containing an event. An event can be anything from
opening a pull request, to merging into master. For our purposes, we are only interested in what are known
as `push` events. These occur every time a branch is pushed, pushed to, or merged.

After deploying the serverless project in this repo, make note of the API Gateway endpoint,

```
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service .zip file to S3 (50.3 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
..............
Serverless: Stack update finished...
Service Information
service: opolis-deployer
stage: prod
region: us-west-2
stack: opolis-deployer-prod
api keys:
  None
endpoints:
  POST - https://xxxxxxx.execute-api.us-west-2.amazonaws.com/prod/webhook <------- *
functions:
  listener: opolis-deployer-prod-listener
  builder: opolis-deployer-prod-builder
  notifier: opolis-deployer-prod-notifier
  s3cleaner: opolis-deployer-prod-s3cleaner
  s3deployer: opolis-deployer-prod-s3deployer
  stack-cleaner: opolis-deployer-prod-stack-cleaner
Serverless: Removing old service versions...
```

On the GitHub repo page, go to "Settings" > "Webhooks" > "Add webhook". Provide the API Gateway endpoint as the hook
destination, the HMAC key set as a build system parameter during setup (`opolis.github.hmac`),
and select "Just the `push` event". Be sure the content type is set to `application/json`.

And that's it! Check out the status updates on each commit pushed to GitHub to track that commit's
progress through the build system. But, first you need to create some files for `opolis/deployer` to use.

## Configuring `opolis/deployer`

`opolis/deployer` expects the following files to be present in the repository when it picks up
a `push` event from GitHub. These files do not have to be present in the `master` branch right away. `opolis/deployer`
will simply see them as part of your branch, allowing you to test the entire deployment lifecycle in isolation
from the rest of the work in the repository. (Yes, that means you can test changes to your build and deploy
configuration in a branch before promoting the changes to `master`!)

The following is a list of files to crate, along with their description. For now, `opolis/deployer` expects all files to live
in a directory at the root of your repo called `deploy`. This may be configurable in the future, but this
setup will always be supported.

Concrete examples of these files can be found in [`examples`](examples.md).

### `deploy/pipeline.json`

**Required**

Defines the entire CI/CD pipeline as a
[CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template. `opolis/deployer`
will pass this template to CloudFormation so it can spin up a unique CodePipeline instance for your project.

This template is *required* to accept the following parameters. If any are missing, the prepartion phase
will fail. These parameters are provided by `opolis/deployer` at runtime, you are not responsible for specifying them,
just including and referencing them in the template.

|Key|Value|
|---|-----|
|`ArtifactStore`|Name of the S3 bucket responsible for storing pipeline artifacts|
|`RepoOwner`|GitHub repo namespace, i.e. `ngmiller`|
|`RepoName`|GitHub repo name|
|`RepoBranch`|Branch name to build|
|`RepoToken`|OAuth token with `repo` scope, be sure to set `"NoEcho": true`|
|`Stage`|Used to reference pipeline parameters `development`, `staging`, or `production`|

### `deploy/pipeline.parameters.json`

**Required**

Defines a set of parameters to configure CodePipeline, and use during a particular _invocation_ of the pipeline.
There are three invocation types, or "stages": `development`, `staging`, and `production`. The `development` invocation
occurs for every branch pushed to the repository, `staging` occurs after a merge into master, and `production` occurs
on every tag. The parameters file is keyed accordingly, and each set of parameters must be a list of objects in the form:

```
{
    "ParameterKey": "...",
    "ParameterValue": "..."
}
```

For example,

```
{
    "development": [
        { "ParameterKey": "foo", "ParameterValue": "1" }
    ],
    "staging": [
        { "ParameterKey": "foo", "ParameterValue": "2" }
    ],
    "production": [
        { "ParameterKey": "foo", "ParameterValue": "3" }
    ]
}
```

This mechanism allows the _pipeline itself_ be configured for a particular stage. Maybe you need to inject a Slack
notification Lambda function into your `production` deployment, but not your `development` ones. These parameters
can be used as toggles in CloudFormation to turn that functionality on or off. Or, maybe you need to use a different
build image for `staging` and `production`. Any configuration releated to the _pipeline_ can be done here.

### `deploy/buildspec.yml`

**Optional** _Required only if you define a CodeBuild step in your `pipeline.json`. Most projects will._

Defines the build steps and commands run inside inside CodeBuild. CodeBuild reads this file, running each command in
sequence, failing the entire build if any command fails. If the build is successful, the artifacts are pushed the S3
bucket specified by `ArtifactStore`, and can be used in a subsequent stage in the pipeline.

### `deploy/deploy.json`

**Optional** _Required only if you need a CloudFormation deployment to occur as part of your service lifecycle._

If the end result of your deployment is a CloudFormation stack creation/update, you can use this file
to define that stack.

`deploy/pipeline.json` will contain a CloudFormation `Deploy` step, which will reference this file as it's
template. Various parameters for each stage can be configured with the files below. If you need to pass an environment
variable through to container instance defined in your `deploy.json`, or configure the CloudFormation stack
in any way, use these files.

They should be structured as JSON like so,

```
{
    "Parameters": {
        "MyParameter": "true",
        "AnotherParameter": "db.m3.medium"
    }
}
```

where `MyParameter` and `AnotherParameter` have already been defined as valid parameters in `deploy.json`

#### `deploy/development.json`

Configures `deploy.json` during a `development` invocation.

#### `deploy/staging.json`

Configures `deploy.json` during a `staging` invocation.

#### `deploy/production.json`

Configures `deploy.json` during a `production` invocation.
