# AWS Static Stack GitHub Action

A composite GitHub Action to deploy your static website to the AWS Edge.

## Getting Started

You should be somewhat familiar with the following technologies:

- AWS CloudFormation (Infrastructure as Code)
- AWS S3 (Object storage)
- AWS CloudFront (CDN & Edge Caching)
- GitHub Actions (CI/CD)

Please also read the following to understand what AWS Credentials you should use: <https://github.com/aws-actions/configure-aws-credentials#credentials>

### Step 1: Define your Stack

Use the provided stack, or provide your own.

### Step 2: Define the Actions YAML

//

### Step 3: Deploy

## Debugging

Check the Action output for logs.

If you need to see more verbose logs you can set `ACTIONS_STEP_DEBUG` to `true` as an Action Secret.

Detailed stack logs can be found in CloudFormation in the AWS Console.

## Related Projects

- [github-action-aws-cloudformation](https://github.com/badsyntax/github-action-aws-cloudformation)
- [github-action-aws-cloudfront](https://github.com/badsyntax/github-action-aws-cloudfront)
- [github-action-aws-s3](https://github.com/badsyntax/github-action-aws-s3)

## License

See [LICENSE.md](./LICENSE.md).
