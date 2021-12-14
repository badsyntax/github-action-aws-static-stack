# AWS Static Stack GitHub Action

[![Test](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/test.yml/badge.svg)](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/test.yml)
[![Deploy](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/deploy.yml/badge.svg)](https://github.com/badsyntax/github-action-aws-static-stack/actions/workflows/deploy.yml)

A composite GitHub Action to deploy your static website to the AWS Edge.

This Action is _opinionated_ meaning it relies on a specific AWS configuration (managed by CloudFormation IaC). It's not likely everyone will agree with default stack configuration or workflows, but I've written the sub-Actions to be generic so you can use the sub-Actions to create your own static deploy pipeline if you require more flexibility. Refer to [`action.yaml`](https://github.com/badsyntax/github-action-aws-static-stack/blob/master/action.yml) for example usage of the various Actions.

## Features

- Preview sites with Pull Request comments
- S3 for origin object storage
- Cloudfront Edge caching
- Separate Root & Preview Cloudfront distributions
  - The Preview distribution uses a Lambda to rewrite the URLs to the correct location in the S3 origin
  - The Root distribution routes directly to the S3 origin and does not use a Lambda, to provide the fastest possible response times
- Flexible stack creation (you provide & own the IaC)

The following GitHub Actions are used:

- [badsyntax/github-action-aws-cloudformation](https://github.com/badsyntax/github-action-aws-cloudformation)
- [badsyntax/github-action-aws-cloudfront](https://github.com/badsyntax/github-action-aws-cloudfront)
- [badsyntax/github-action-aws-s3](https://github.com/badsyntax/github-action-aws-s3)
- [badsyntax/github-action-issue-comment](https://github.com/badsyntax/github-action-issue-comment)

## Getting Started

You should be somewhat familiar with the following technologies:

- AWS CloudFormation (Infrastructure as Code)
- AWS S3 (Object storage)
- AWS CloudFront (CDN & Edge Caching)
- GitHub Actions (CI/CD)

Please also read the following to understand what AWS Credentials you should use: <https://github.com/aws-actions/configure-aws-credentials#credentials>

### Step 1: Define your Stack

Use the [provided CloudFormation stack](https://github.com/badsyntax/github-action-aws-static-stack/blob/master/cloudformation/cloudformation-s3bucket-stack.yml) and place it somewhere in your repo, for example in location `./cloudformation/cloudformation-s3bucket-stack.yml`

You are welcome to change the stack as long as you don't change the following outputs:

- `S3BucketName`
- `CFDistributionRootId`
- `CFDistributionPreviewId`

### Step 2: Define your PR Comment Template

Use the [provided pull request comment template](https://github.com/badsyntax/github-action-aws-static-stack/blob/master/.github/pr-comment-template.hbs) and place it somewhere in your repo, for example in location `.github/pr-comment-template.hbs`

### Step 3: Define the Actions YAML

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
          cf-stack-name: 'github-action-example-aws-static-stack'
          cf-template: './cloudformation/cloudformation-s3bucket-stack.yml'
          cf-apply-change-set: ${{ github.event_name != 'repository_dispatch' }}
          token: ${{ secrets.GITHUB_TOKEN }}
          aws-region: 'us-east-1'
          s3-bucket-name: 'github-action-example-aws-static-stack-us-east-1'
          s3-allowed-origins: 'https://example.com, https://*.preview.example.com'
          cloudfront-root-hosts: 'example.com'
          cloudfront-preview-hosts: '*.preview.example.com'
          cloudfront-default-root-object: 'index'
          certificate-arn: 'arn:aws:acm:us-east-1:0001112222:certificate/1234abc-1234-1234-abcd-12345'
          src-dir: './out'
          static-files-glob: 'css/**'
          lambda-version: '1.0.0'
          delete-preview-site-on-pr-close: true
          comment-template: '.github/comment-template.md'
```

### Step 4: Deploy

Send a pull request to your repository to create the stack and deploy a preview site.

Note that `cf-apply-change-set` must be set to `true` to allow the stack to be created before attempting a deploy. The only time you don't want to set this is when the job is run via webhook (eg `repository_dispatch`).

### Step 5: Adjust Domain Records

Create `CNAME` records that point to the relevant distribution.

For example I have the following records defined:

```console
static-example CNAME 1234abcd.cloudfront.net.
*.preview.static-example CNAME 5678edfgh.cloudfront.net.
```

View more info on using a custom domain: <https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html#CreatingCNAME>

## Action Inputs

All of the following inputs are required:

| Name                              | Description                                                                                      | Example                                                                    |
| --------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| `cf-stack-name`                   | The name of the Cloudformation stack to be created                                               | `example-com-static-cloudformation-stack`                                  |
| `cf-template`                     | The relative path to the CloudFormation stack template                                           | `./cloudformation/s3bucket_with_cloudfront.yml`                            |
| `cf-apply-change-set`             | Whether to apply the CloudFormation ChangeSet (if any)                                           | `true`                                                                     |
| `token`                           | GitHub Token used for commenting on Pull Requests                                                | `${{ secrets.GITHUB_TOKEN }}`                                              |
| `aws-region`                      | 'The AWS region in which to create the stack. You should set this to `us-east-1`                 | `us-east-1`                                                                |
| `s3-bucket-name`                  | The name of S3 bucket to be created, to store your static files. Must end with region name       | `example.com-us-east-1`                                                    |
| `s3-allowed-origins`              | A list of allowed domains to request resources from S3                                           | `https://example.com,https://*.preview.example.com`                        |
| `cloudfront-root-hosts`           | A list of hosts assigned to the Root CloudFront distribution                                     | `example.com`                                                              |
| `cloudfront-preview-hosts`        | A list of hosts assigned to the Preview CloudFront distribution                                  | `*.preview.example.com`                                                    |
| `cloudfront-default-root-object`  | The CloudFront default root object                                                               | `index`                                                                    |
| `certificate-arn`                 | ARN of the certificate for the root and preview domains                                          | `arn:aws:acm:us-east-1:1234567:certificate/123abc-123abc-1234-5678-abcdef` |
| `src-dir`                         | Path to build/out directory that contains the static files                                       | `./out`                                                                    |
| `static-files-glob`               | Glob pattern for immutable static files                                                          | `_next/**`                                                                 |
| `lambda-version`                  | The lambda version. Required to deploy a new lambda. You must update this if changing the lambda | `1.0.0`                                                                    |
| `delete-preview-site-on-pr-close` | Whether to delete the preview site on PR close                                                   | `true`                                                                     |
| `comment-template`                | Path to the Pull Request comment template                                                        | `.github/comment-template.md`                                              |

## Action Outputs

| Name           | Description                                                               | Example                   |
| -------------- | ------------------------------------------------------------------------- | ------------------------- |
| `modifiedKeys` | A comma separated list of modified object keys (either synced or removed) | `file1,folder1/file2.ext` |

## Debugging

Check the Action output for logs.

If you need to see more verbose logs you can set `ACTIONS_STEP_DEBUG` to `true` as an Action Secret.

Detailed stack logs can be found in CloudFormation in the AWS Console.

## License

See [LICENSE.md](./LICENSE.md).
