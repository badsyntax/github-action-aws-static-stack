# AWS Static Stack GitHub Action

[![Test](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/test.yml/badge.svg)](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/test.yml)
[![Deploy](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/deploy.yml/badge.svg)](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/deploy.yml)

A composite GitHub Action to deploy your static website to the AWS Edge.

The following GitHub Actions are used:

- [badsyntax/github-action-aws-cloudformation](https://github.com/badsyntax/github-action-aws-cloudformation)
- [badsyntax/github-action-aws-cloudfront](https://github.com/badsyntax/github-action-aws-cloudfront)
- [badsyntax/github-action-aws-s3](https://github.com/badsyntax/github-action-aws-s3)

## Getting Started

You should be somewhat familiar with the following technologies:

- AWS CloudFormation (Infrastructure as Code)
- AWS S3 (Object storage)
- AWS CloudFront (CDN & Edge Caching)
- GitHub Actions (CI/CD)

Please also read the following to understand what AWS Credentials you should use: <https://github.com/aws-actions/configure-aws-credentials#credentials>

### Step 1: Define your Stack

Use the provided stack and place it somewhere in your repo, for example in location `./cloudformation/cloudformation-s3bucket-stack.yml`.

You are welcome to change the stack as long as you don't change the following outputs:

- `S3BucketName`
- `CFDistributionRootId`
- `CFDistributionPreviewId`

### Step 2: Define the Actions YAML

```yaml
name: 'Deploy'

concurrency:
  group: prod_deploy
  cancel-in-progress: false

on:
  repository_dispatch:
  workflow_dispatch:
  pull_request:
    types: [opened, synchronize, reopened, closed]
  push:
    branches:
      - master

jobs:
  deploy:
    name: 'Deploy'
    runs-on: ubuntu-20.04
    if: github.actor != 'dependabot[bot]' && (github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository)
    steps:
      - uses: actions/checkout@v2

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - uses: badsyntax/github-action-aws-static-stack@v0.0.1
        name: Deploy Site
        with:
          cfStackName: 'github-action-example-aws-static-stack'
          cfTemplate: './cloudformation/cloudformation-s3bucket-stack.yml'
          cfApplyChangeSet: ${{ github.event_name != 'repository_dispatch' }}
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
          awsRegion: 'us-east-1'
          s3BucketName: 'github-action-example-aws-static-stack-us-east-1'
          s3AllowedOrigins: 'https://example.com, https://*.preview.example.com'
          cloudFrontRootHosts: 'example.com'
          cloudFrontPreviewHosts: '*.preview.example.com'
          cloudFrontDefaultRootObject: 'index.html'
          certificateARN: 'arn:aws:acm:us-east-1:0001112222:certificate/1234abc-1234-1234-abcd-12345'
          srcDir: './out'
          staticFilesGlob: 'css/**'
```

### Step 3: Deploy

Send a pull request to your repository to create the stack and deploy a preview site.

Note that `cfApplyChangeSet` must be set to `true` to allow the stack to be created before attempting a deploy. The only time you don't want to set this is when the job is run via webhook (eg `repository_dispatch`).

### Step 4: Adjust Domain Records

Create `CNAME` records that point to the relevant distribution.

For example I have the following records defined:

```console
static-example CNAME 1234abcd.cloudfront.net.
*.preview.static-example CNAME 5678edfgh.cloudfront.net.
```

View more info on using a custom domain: <https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html#CreatingCNAME>

## Debugging

Check the Action output for logs.

If you need to see more verbose logs you can set `ACTIONS_STEP_DEBUG` to `true` as an Action Secret.

Detailed stack logs can be found in CloudFormation in the AWS Console.

## License

See [LICENSE.md](./LICENSE.md).
