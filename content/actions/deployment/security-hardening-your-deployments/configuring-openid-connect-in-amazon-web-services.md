---
title: Configuring OpenID Connect in Amazon Web Services
shortTitle: OpenID Connect in AWS
intro: Use OpenID Connect within your workflows to authenticate with Amazon Web Services.
versions:
  fpt: '*'
  ghec: '*'
  ghes: '*'
type: tutorial
topics:
  - Security
---
 
{% data reusables.actions.enterprise-github-hosted-runners %}

## Overview

OpenID Connect (OIDC) allows your {% data variables.product.prodname_actions %} workflows to access resources in Amazon Web Services (AWS), without needing to store the AWS credentials as long-lived {% data variables.product.prodname_dotcom %} secrets.

This guide explains how to configure AWS to trust {% data variables.product.prodname_dotcom %}'s OIDC as a federated identity, and includes a workflow example for the [`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials) that uses tokens to authenticate to AWS and access resources.

## Prerequisites

{% data reusables.actions.oidc-link-to-intro %}

{% data reusables.actions.oidc-security-notice %}

## Adding the identity provider to AWS

To add the {% data variables.product.prodname_dotcom %} OIDC provider to IAM, see the [AWS documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html).

{% ifversion ghec %}
- Two intermediate Certificate Authority(CA) certificate thumbprints must be added to the GitHub Actions OIDC IdP provider:
  - `6938fd4d98bab03faadb97b34396831e3780aea1`
  - `1c58a3a8518e8759bf075b76b750d4f2df264fcd`
{% endif %}
- For the provider URL: Use {% ifversion ghes %}`https://HOSTNAME/_services/token`{% else %}`https://token.actions.githubusercontent.com`{% endif %}
- For the "Audience": Use `sts.amazonaws.com` if you are using the [official action](https://github.com/aws-actions/configure-aws-credentials).

### Configuring the role and trust policy

To configure the role and trust in IAM, see the AWS documentation for ["Assuming a Role"](https://github.com/aws-actions/configure-aws-credentials#assuming-a-role) and ["Creating a role for web identity or OpenID connect federation"](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html).

Edit the trust policy to add the `sub` field to the validation conditions. For example:

```json copy
"Condition": {
  "StringEquals": {
    "{% ifversion ghes %}HOSTNAME/_services/token{% else %}token.actions.githubusercontent.com{% endif %}:aud": "sts.amazonaws.com",
    "{% ifversion ghes %}HOSTNAME/_services/token{% else %}token.actions.githubusercontent.com{% endif %}:sub": "repo:octo-org/octo-repo:ref:refs/heads/octo-branch"
  }
}
```

In the following example, `StringLike` is used with a wildcard operator (`*`) to allow any branch, pull request merge branch, or environment from the `octo-org/octo-repo` organization and repository to assume a role in AWS.

```json copy
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::123456123456:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:octo-org/octo-repo:*"
                },
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

## Updating your {% data variables.product.prodname_actions %} workflow

To update your workflows for OIDC, you will need to make two changes to your YAML:
1. Add permissions settings for the token.
1. Use the [`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials) action to exchange the OIDC token (JWT) for a cloud access token.

### Adding permissions settings

 {% data reusables.actions.oidc-permissions-token %}

### Requesting the access token

The `aws-actions/configure-aws-credentials` action receives a JWT from the {% data variables.product.prodname_dotcom %} OIDC provider, and then requests an access token from AWS. For more information, see the AWS [documentation](https://github.com/aws-actions/configure-aws-credentials).

- `<example-bucket-name>`: Add the name of your S3 bucket here.
- `<role-to-assume>`: Replace the example with your AWS role.
- `<example-aws-region>`: Add the name of your AWS region here.

```yaml copy
# Sample workflow to access AWS resources when workflow is tied to branch
# The workflow Creates static website using aws s3
name: AWS example workflow
on:
  push
env:
  BUCKET_NAME : "<example-bucket-name>"
  AWS_REGION : "<example-aws-region>"
# permission can be added at job level or workflow level    
permissions:
      id-token: write   # This is required for requesting the JWT
      contents: read    # This is required for actions/checkout
jobs:
  S3PackageUpload:
    runs-on: ubuntu-latest
    steps:
      - name: Git clone the repository
        uses: {% data reusables.actions.action-checkout %}
      - name: configure aws credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::1234567890:role/example-role
          role-session-name: samplerolesession
          aws-region: {% raw %}${{ env.AWS_REGION }}{% endraw %}
      # Upload a file to AWS s3
      - name:  Copy index.html to s3
        run: |
          aws s3 cp ./index.html s3://{% raw %}${{ env.BUCKET_NAME }}{% endraw %}/
```
