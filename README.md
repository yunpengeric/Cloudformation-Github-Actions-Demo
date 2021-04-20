# Using GitHub Actions to Deploy CloudFormation Stacks

# Background

GitHub Actions is generally available at the end of 2019. Thanks to the open marketplace of GitHub Actions, more and more projects are using it. 

Whether you are in the Devs or Ops team, you must be familiar with infrastructure as code. However, if you don't have a good pipeline, the deployment process can be a headache.

In this article, you will learn how to use GitHub Actions to deploy CloudFormation stacks.

# Prerequisites

- A GitHub Account (Click [here](https://docs.github.com/en/github/setting-up-and-managing-billing-and-payments-on-github/about-billing-for-github-actions) to see free tier information)
- A Programmatic Access IAM user in AWS
- CloudFormation & S3 permissions

# Process

## Create CFN Templates

Let us create a simple CloudFormation template for creating an S3 bucket.

The filename is `s3.yml`.

```yaml
  Type: AWS::S3::Bucket
    Properties:
      AccessControl: Private
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
```

Then we would like to name our own S3 bucket, and for our template to be reusable, we want to pass parameters to CloudFormation stacks.

```yaml
Parameters:
  BucketName:
    Description: Name your Bucket
    Type: String

Resources:
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: Private
      BucketName: !Ref BucketName
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
```

Finally, we can add descriptions and format version and checkin the template in our repository.  Click [here](https://github.com/yunpengeric/Cloudformation-Github-Actions-Demo/blob/main/s3.yml) to view the full template.

## Create GitHub Workflows

Create `.github/workflows/main.yml` under our git root directory.  Let us write our first worflow.

### Configure AWS Credentials

We are using this action to configure AWS credentials.

["Configure AWS Credentials" Action For GitHub Actions - GitHub Marketplace](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions)

We can write the step according to the README.

```yaml
- name: Configure AWS Credentials
  id: creds
  uses: aws-actions/configure-aws-credentials@v1
  with:
     aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
     aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
     aws-region: "ap-southeast-2"
```

In AWS IAM console, get your access key ID and secret access key.

[AWS Account and Access Keys](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html)

Then navigate to the main page of  GitHub repository > Click `Setting` > In the left sidebar, click `Secrets` > Click `New repository secret` >  Create `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

![AccessKeyImage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bh74157kvi8e4k7bh6lc.png)

### Create Deployment Actions

We are using this action to deploy CFN stacks. 

[AWS CloudFormation "Deploy CloudFormation Stack" Action for GitHub Actions - GitHub Marketplace](https://github.com/marketplace/actions/aws-cloudformation-deploy-cloudformation-stack-action-for-github-actions)

We can write the step according to the README.

- As I am writing this article, the latest version is `v1.0.3`.

```yaml
- name: Deploy to AWS CloudFormation
  uses: aws-actions/aws-cloudformation-github-deploy@v1.0.3
  with:
    name: S3Bucket
    template: s3.yml
```

We also need to pass the input parameter `BucketName`.

```yaml
# Controls when the action will run.
on:
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
    inputs:
      BucketName:
        description: "Name your Bucket"
        required: true
	default: "demo-bucket"
```

Then we add `parameter-overrides` in the `Deploy to AWS CloudFormation` step.

```yaml
- name: Deploy S3 Buckets CloudFormation Stacks
  uses: aws-actions/aws-cloudformation-github-deploy@v1.0.3
  with:
    name: s3-buckets
    template: s3.yml
    parameter-overrides: >-
    	BucketName=${{ github.event.inputs.bucketName }}
```

Let's finally organise our action file `main.yml`.

```yaml
name: Deploy CloudFormation Stacks

# Controls when the action will run.
on:
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
    inputs:
      region:
        description: "AWS Region"
        required: true
        default: "ap-southeast-2"
      bucketName:
        description: "S3 Bucket Name"
        required: true

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  cfn-deployment:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        
      - name: Configure AWS credentials
        id: creds
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ github.event.inputs.region }} 

      - name: Deploy S3 Buckets CloudFormation Stacks
        id: s3-buckets
        uses: aws-actions/aws-cloudformation-github-deploy@v1.0.3
        with:
          name:  s3-buckets
          template: s3.yml
          parameter-overrides: >-
            BucketName=${{ github.event.inputs.bucketName }}
```

## Use GitHub Workflows

Make sure you have pushed all the files. Just like the following repo.

[yunpengeric/Cloudformation-Github-Actions-Demo](https://github.com/yunpengeric/Cloudformation-Github-Actions-Demo)

Then navigate to the main page of  GitHub repository > Click `Actions` > In the left sidebar, click `Deploy CloudFormation Stacks` >  Click `Run Workflow` .

You should see the following pre-filled data > Enter your S3 bucket name (Please note an S3 bucket name is globally unique) > Run Workflow

![actionsimage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7wj9nby1ca23s3ucdohn.png)

After a minute or so, you will see that our S3 bucket has been successfully deployed. If you see any errors, please leverage GitHub logs and CloudFormation console to troubleshoot. 

# Conclusion

We are still far from it if our goal is to create a good pipeline in production environment. You should take these things into consideration as well.

- How you can rotate keys to ensure security.
- How to modify your CFN templates so that when you deploy a second bucket, you don't delete the first one.
