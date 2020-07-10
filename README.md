# CDK Workshop

Follow through the workshop at https://cdkworkshop.com/20-typescript.html

## Personal notes

- Why is the directory with the sources called `bin`?
- The first time you deploy an AWS CDK app into an environment (account/region), you’ll need to install a "bootstrap stack". This stack includes resources that are needed for the toolkit’s operation. For example, the stack includes an S3 bucket that is used to store templates and assets during the deployment process
  - Currently, the following resources are created
    - `StagingBucket cdktoolkit-stagingbucket-18dnewnexgqtp AWS::S3::Bucket` => no versionning, encrypted using KMS
    - `StagingBucketPolicy CDKToolkit-StagingBucketPolicy-LFATUNLTPE3U AWS::S3::BucketPolicy` => content below
  - There is no CLI option to destroy bootstrapped resources, no `cdk bootstrap --destroy`: should you stop using CDK you should delete the CF stack yourself

```
{
    "Version": "2012-10-17",
    "Id": "AccessControl",
    "Statement": [
        {
            "Sid": "AllowSSLRequestsOnly",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::cdktoolkit-stagingbucket-18dnewnexgqtp",
                "arn:aws:s3:::cdktoolkit-stagingbucket-18dnewnexgqtp/*"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```

- IAM policy may be auto-generated, but not always: e.g. a lambda uses a DynamoDB client to `UpdateItem`, you should grant the required permission yourself
  - It's a step back from other AWS frameworks for developpers (like Chalice) which are making a better job auto-generating these policies
- CDK `deploy` asks human confirmation before applying change to IAM policy: `This deployment will make potentially sensitive changes...`
  - Can be bypassed by `--require-approval=never"` for CI usage (e.g. for use from the CD pipeline)
- Generated name are downright ugly: `CdkWorkshopStack-CdkWorkshopTopicD368A42F-78K64F9UIAXW`
  - first hash `D368A42F` is added by CDK for unicity and second hash `78K64F9UIAXW` by CloudFormation when creating the resource
  - it's due to CF limitations, see https://github.com/aws/aws-cdk/issues/1424
  - fortunately, logical ID may be overridden using `overrideLogicalId(...)` on the resource
- Outputs from constructs are displayed after executing `cdk deploy`

```
cdk deploy
CdkWorkshopStack: deploying...

 ✅  CdkWorkshopStack (no changes)

Outputs:
CdkWorkshopStack.Endpoint8024A810 = https://xxxxxxxxx.execute-api.eu-west-3.amazonaws.com/prod/

Stack ARN:
arn:aws:cloudformation:eu-west-3:476440821995:stack/CdkWorkshopStack/af431b90-c2a3-11ea-b0cd-0a09d4f4622c
```

- Note that these outputs are not easily parsable from a tool
- You cannot output to a better format directly but you can use the `--outputs-file` option to print the outputs as json in a file

```
cdk deploy --outputs-file=output.json
```

```
cat output.json
{
  "CdkWorkshopStack": {
    "Endpoint8024A810": "https://xxxxxxxxx.execute-api.eu-west-3.amazonaws.com/prod/"
  }
}
```

- Even in JSON, the output is not so automation-friendly as the generated ID are random and there is no indication of type or other properties
- In the end, the classic scenario of getting stack information from CDK to inject them into an application configuration (e.g. to inject the name of a DynamoDB table used inside the app code built as a container) is not straightforward
