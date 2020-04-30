cloudsplaining
--------------

`cloudsplaining` is an AWS IAM Security Assessment tool that identifies violations of least privilege and generates a risk-prioritized HTML report with a triage worksheet.

[![Build Status](https://travis-ci.com/salesforce/cloudsplaining.svg?branch=master)](https://travis-ci.com/salesforce/cloudsplaining)
[![Documentation Status](https://readthedocs.org/projects/cloudsplaining/badge/?version=latest)](https://cloudsplaining.readthedocs.io/en/latest/?badge=latest)
[![Join the chat at https://gitter.im/cloudsplaining](https://badges.gitter.im/cloudsplaining.svg)](https://gitter.im/cloudsplaining?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)


* [Example report](https://opensource.salesforce.com/cloudsplaining/)

> ![](https://github.com/salesforce/cloudsplaining/raw/master/docs/_images/cloudsplaining-report.gif)

## Documentation

For full documentation, please visit the [project on ReadTheDocs](https://cloudsplaining.readthedocs.io/en/latest/).

* [Installation](#installation)
* [Cheat sheet](#cheatsheet)
* [Example report](https://opensource.salesforce.com/cloudsplaining/)

## Overview

Cloudsplaining identifies violations of least privilege in AWS IAM policies and generates a pretty HTML report with a triage worksheet. It can scan all the policies in your AWS account or it can scan a single policy file.

It helps to identify IAM actions that do not leverage resource constraints. It also helps prioritize the remediation process by flagging IAM policies that present the following risks to the AWS account in question without restriction:
* Data Exfiltration (`s3:GetObject`, `ssm:GetParameter`, `secretsmanager:GetSecretValue`)
* Infrastructure Modification
* Resource Exposure (the ability to modify resource-based policies)
* Privilege Escalation (based on Rhino Security Labs research)

You can also specify a custom exclusions file to filter out results that are False Positives for various reasons. For example, User Policies are permissive by design, whereas System roles are generally more restrictive. You might also have exclusions that are specific to your organization's multi-account strategy or AWS application architecture.


## Motivation

[Policy Sentry](https://engineering.salesforce.com/salesforce-cloud-security-automating-least-privilege-in-aws-iam-with-policy-sentry-b04fe457b8dc) revealed to us that it is possible to finally write IAM policies according to least privilege in a scalable manner. Before Policy Sentry was released, it was too easy to find IAM policy documents that lacked resource constraints. Consider the policy below, which allows the IAM principal (a role or user) to run `s3:PutObject` on any S3 bucket in the AWS account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "*"
    }
  ]
}
```

This is bad. Ideally, access should be restricted according to resource ARNs, like so:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

Policy Sentry [makes it really easy to do this](https://github.com/salesforce/policy_sentry/#policy-sentry). Once Infrastructure as Code developers or AWS Administrators gain familiarity with the tool (which is quite easy to use), we've found that adoption starts very quickly. **However**, if you've been using AWS, there is probably a very large backlog of IAM policies that could use an uplift. If you have hundreds of AWS accounts with dozens of policies in each, how can we lock down those AWS accounts by programmatically identifying the policies that should be fixed?

That's why we wrote Cloudsplaining.

Cloudsplaining identifies violations of least privilege in AWS IAM policies and generates a pretty HTML report with a triage worksheet. It can scan all the policies in your AWS account or it can scan a single policy file.

## Installation

* Homebrew

```bash
brew tap salesforce/cloudsplaining https://github.com/salesforce/cloudsplaining
brew install cloudsplaining
```

* Pip3

```bash
pip3 install --user cloudsplaining
```

* Now you should be able to execute `cloudsplaining` from command line by running `cloudsplaining --help`.


### Scanning an entire AWS Account

#### Downloading Account Authorization Details

We can scan an entire AWS account and generate reports. To do this, we leverage the AWS IAM [get-account-authorization-details](https://docs.aws.amazon.com/cli/latest/reference/iam/get-account-authorization-details.html) API call, which downloads a large JSON file (around 100KB per account) that contains all of the IAM details for the account. This includes data on users, groups, roles, customer-managed policies, and AWS-managed policies.

* To do this, set your AWS access keys as environment variables:

```bash
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
# If you are using MFA or STS; optional but highly recommended
export AWS_SESSION_TOKEN=...
```

* Then run `cloudsplaining`'s `download` command:

```bash
cloudsplaining download
```

* If you prefer to use your `~/.aws/credentials` file instead of environment variables, you can specify the profile name:

```bash
cloudsplaining download --profile default
```

It will download a file titled `default.json` in your current directory.

#### Create Exclusions file

Cloudsplaining tool does not attempt to understand the context behind everything in your AWS account. It's possible to understand the context behind some of these things programmatically - whether the policy is applied to an instance profile, whether the policy is attached, whether inline IAM policies are in use, and whether or not AWS Managed Policies are in use. **Only you know the context behind the design of your AWS infrastructure and the IAM strategy**.

As such, it's important to eliminate False Positives that are context-dependent. You can do this with an exclusions file. We've included a command that will generate an exclusions file for you so you don't have to remember the required format.

You can create an exclusions template via the following command:

```bash
cloudsplaining create-exclusions-file
```

This will generate a file in your current directory titled `exclusions.yml`.

Now when you run the `scan` command, you can use the exclusions file like this:

```bash
cloudsplaining scan --exclusions-file exclusions.yml --file examples/files/example.json --output examples/files/
```

For more information on the structure of the exclusions file, see [Filtering False Positives](#filtering-false-positives)

#### Scanning the Authorization Details file

Now that we've downloaded the account authorization file, we can scan *all* of the AWS IAM policies with `cloudsplaining`.

Run the following command:

```bash
cloudsplaining scan --exclusions-file exclusions.yml --file examples/files/example.json --output examples/files/
```

It will create an HTML report like this:

![](docs/_images/cloudsplaining-report.gif)


It will also create a raw JSON data file:

* `default-iam-results.json`: This contains the raw JSON output of the report. You can use this data file for operating on the scan results for various purposes. For example, you could write a Python script that parses this data and opens up automated JIRA issues or Salesforce Work Items. An example entry is shown below. The full example can be viewed at [examples/output/example-authz-details-results.json](examples/files/iam-results-example.json)

```json
{
    "example-authz-details": [
        {
            "AccountID": "012345678901",
            "ManagedBy": "Customer",
            "PolicyName": "InsecureUserPolicy",
            "Arn": "arn:aws:iam::012345678901:user/userwithlotsofpermissions",
            "ActionsCount": 2,
            "ServicesCount": 1,
            "Actions": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Services": [
                "s3"
            ]
        }
    ]
}
```


See the [examples/files](examples/files) folder for sample output.

#### Filtering False Positives

Resource constraints are best practice - especially for system roles/instance profiles - but sometimes, these are by design. For example, consider a situation where a custom IAM policy is used on an instance profile for an EC2 instance that provisions Terraform. *In this case, broad permissions are design requirements* - so we don't want to include these in the results.

You can create an exclusions template via the following command:

```bash
cloudsplaining create-exclusions-file
```

This will generate a file in your current directory titled `exclusions.yml`.

The default exclusions file looks like this:

```yaml
# Policy names to exclude from evaluation
# Suggestion: Add policies here that are known to be overly permissive by design, after you run the initial report.
policies:
  - "AWSServiceRoleFor*"
  - "*ServiceRolePolicy"
  - "*ServiceLinkedRolePolicy"
  - "AdministratorAccess" # Otherwise, this will take a long time
  - "service-role*"
  - "aws-service-role*"
# Don't evaluate these roles, users, or groups as part of the evaluation
roles:
  - "service-role*"
  - "aws-service-role*"
users:
  - ""
groups:
  - ""
# Read-only actions to include in the results, such as s3:GetObject
# By default, it includes Actions that could lead to Data Leaks
include-actions:
  - "s3:GetObject"
  - "ssm:GetParameter"
  - "ssm:GetParameters"
  - "ssm:GetParametersByPath"
  - "secretsmanager:GetSecretValue"
# Write actions to include from the results, such as kms:Decrypt
exclude-actions:
  - ""
```

* Make any additions or modifications that you want.
  * Under `policies`, list the path of policy names that you want to exclude.
  * If you want to exclude a role titled `MyRole`, list `MyRole` or `MyR*` in the `roles` list.
  * You can follow the same approach for `users` and `groups` list.

Now when you run the `scan` command, you can use the exclusions file like this:

```bash
cloudsplaining scan --exclusions-file exclusions.yml --file examples/files/example.json --output examples/files/
```


### Scanning a single policy

You can also scan a single policy file to identify risks instead of an entire account.

```bash
cloudsplaining scan-policy-file --file examples/policies/explicit-actions.json
```

The output will include a finding description and a list of the IAM actions that do not leverage resource constraints.

The output will resemble the following:

```console
Issue found: Data Exfiltration
Actions: s3:GetObject

Issue found: Resource Exposure
Actions: ecr:DeleteRepositoryPolicy, ecr:SetRepositoryPolicy, s3:BypassGovernanceRetention, s3:DeleteAccessPointPolicy, s3:DeleteBucketPolicy, s3:ObjectOwnerOverrideToBucketOwner, s3:PutAccessPointPolicy, s3:PutAccountPublicAccessBlock, s3:PutBucketAcl, s3:PutBucketPolicy, s3:PutBucketPublicAccessBlock, s3:PutObjectAcl, s3:PutObjectVersionAcl

Issue found: Unrestricted Infrastructure Modification
Actions: ecr:BatchDeleteImage, ecr:CompleteLayerUpload, ecr:CreateRepository, ecr:DeleteLifecyclePolicy, ecr:DeleteRepository, ecr:DeleteRepositoryPolicy, ecr:InitiateLayerUpload, ecr:PutImage, ecr:PutImageScanningConfiguration, ecr:PutImageTagMutability, ecr:PutLifecyclePolicy, ecr:SetRepositoryPolicy, ecr:StartImageScan, ecr:StartLifecyclePolicyPreview, ecr:TagResource, ecr:UntagResource, ecr:UploadLayerPart, s3:AbortMultipartUpload, s3:BypassGovernanceRetention, s3:CreateAccessPoint, s3:CreateBucket, s3:DeleteAccessPoint, s3:DeleteAccessPointPolicy, s3:DeleteBucket, s3:DeleteBucketPolicy, s3:DeleteBucketWebsite, s3:DeleteObject, s3:DeleteObjectTagging, s3:DeleteObjectVersion, s3:DeleteObjectVersionTagging, s3:GetObject, s3:ObjectOwnerOverrideToBucketOwner, s3:PutAccelerateConfiguration, s3:PutAccessPointPolicy, s3:PutAnalyticsConfiguration, s3:PutBucketAcl, s3:PutBucketCORS, s3:PutBucketLogging, s3:PutBucketNotification, s3:PutBucketObjectLockConfiguration, s3:PutBucketPolicy, s3:PutBucketPublicAccessBlock, s3:PutBucketRequestPayment, s3:PutBucketTagging, s3:PutBucketVersioning, s3:PutBucketWebsite, s3:PutEncryptionConfiguration, s3:PutInventoryConfiguration, s3:PutLifecycleConfiguration, s3:PutMetricsConfiguration, s3:PutObject, s3:PutObjectAcl, s3:PutObjectLegalHold, s3:PutObjectRetention, s3:PutObjectTagging, s3:PutObjectVersionAcl, s3:PutObjectVersionTagging, s3:PutReplicationConfiguration, s3:ReplicateDelete, s3:ReplicateObject, s3:ReplicateTags, s3:RestoreObject, s3:UpdateJobPriority, s3:UpdateJobStatus

```


## Cheatsheet

```bash
# Download authorization details
cloudsplaining download
# Download from a specific profile
cloudsplaining download --profile someprofile
# Download authorization details for **all** of your AWS profiles
cloudsplaining download --profile all

# Scan Authorization details
cloudsplaining scan --file default.json
# Scan Authorization details with custom exclusions
cloudsplaining scan --file default.json --exclusions-file exclusions.yml

# Scan Policy Files
cloudsplaining scan-policy-file --file examples/policies/wildcards.json
cloudsplaining scan-policy-file --file examples/policies/wildcards.json  --exclusions-file examples/example-exclusions.yml
```

## FAQ

**Will it scan all policies by default?**

No, it will only scan policies that are attached to IAM principals.

**Will the download command download all policy versions?**

Not by default. If you want to do this, specify the `--include-non-default-policy-versions` flag. Note that the `scan` tool does not currently operate on non-default versions.

**I followed the installation instructions but can't execute the program via command line. What do I do?**

This is likely an issue with your PATH. Your PATH environment variable is not considering the binary packages installed by `pip3`. On a Mac, you can likely fix this by entering the command below, depending on the versions you have installed. YMMV.

```bash
export PATH=$HOME/Library/Python/3.7/bin/:$PATH
```

## References

* [Policy Sentry](https://github.com/salesforce/policy_sentry/) by [Kinnaird McQuade](https://twitter.com/kmcquade3) at Salesforce
* [Parliament](https://github.com/duo-labs/parliament/) by [Scott Piper](https://twitter.com/0xdabbad00) at Summit Route
* AWS Privilege Escalation Methods by Spencer Gietzen at Rhino Security Labs
* [Understanding Access Level Summaries within Policy Summaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_understand-policy-summary-access-level-summaries.html)