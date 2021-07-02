# Jekyll Site Hosted on AWS S3 using Github Actions

_Last updated: July 01, 2021_

## Overview
The purpose of this runbook is to define the necessary steps in order to deploy a secure static website hosted on an AWS S3 Bucket, served by an AWS CloudFront Distribution (CDN) which will all be automed using a GitHub Action Workflow.

Our use case for this runbook will be following the build of my personal resume website: [jennasprattler.com](https://jennasprattler.com/) or [www.jennasprattler.com](https://www.jennasprattler.com/)

### Pre-requisites
- Static website
  - Check here for a curated directory of [Jekyll Themes](https://jekyllthemes.io/)
  - I'm using this Jekyll theme for my website: [modern-resume-theme](https://github.com/sproogen/modern-resume-theme)
- GitHub Account and Repo 
  - To store your website files and run GitHub Actions Workflows
- AWS Account
  - To implement Route 53 DNS records, S3 Bucket storage, CloudFront CDN and AWS Certificate Manager for SSL Certificates

### High-level Overview - Traffic Flow
![jekyll-web-flow](/images/jekyll-web-flow.jpg)
- Mobile and Desktop broswers are supported for the Jekyll website
- AWS Route 53 performs DNS resolution for [jennasprattler.com](https://jennasprattler.com/) or [www.jennasprattler.com](https://www.jennasprattler.com/)
- AWS CloudFront CDN serves up any cached content of your website from one of its Edge Locations closest to you
- AWS Certificate Manager hosts the SSL Certificate for your website which gets assigned to your CloudFront CDN encrypting all user traffic
- AWS S3 Bucket hosts all of your website files where the CloudFront CDN retrieves anything that it doesn't have cached

### High-level Overview - CI/CD Flow
![jekyll-code-deploy](/images/jekyll-code-deploy.jpg)
- Developer writes the code to display on the Jekyll Website
- GitHub hosts the repo for the Jekyll website files
- GitHub Actions kickoff the CI/CD Workflow whenever a push is made to the main branch
- CI/CD Workflow uploads the output files from the Jekyll _site directory to the AWS S3 bucket
- CloudFront invalidation is run in order to clear out any cached content and immediately serve up the new/updated S3 website content

### Procedure
#### Create DNS Records
I'll be creating DNS Records using Route 53 however, you can use another domain provider if you like just note that you'll need to follow a slightly different procedure then what I've defined below.
1. Navigate to the AWS Route 53 service and check the availability of your domain name - if available purchase it. At the time of this writing it costs $12 per year for a new domain.
2. It will take approximately 30 minutes for your new domain to be registered so either take a break here or go ahead and get started with creating the S3 bucket in the next section. Come back here to complete setting up your public hosted zone.
3. Setup your hosted zone by entering your new Domain Name and selecting Public Hosted Zone in the Type dropdown.
4. 

#### Create S3 Bucket
- Copy the Cloudformation stack YAML code below and replace all FIXME values per your environment.
  - `BucketName` must match your domain name exactly
  - `PublicAccessBlockConfiguration` properties are all set to false in order to allow public access to your website
  - `BucketPolicy` has `s3:GetObject` action set to allow so that anyone can read the object data and view the website
  - `WebsiteConfiguration` enables the static website capability in S3 
  - `WWWBucket` creates an empty bucket only used to redirect www.FIXME.com traffic to your FIXME.com bucket and is only needed if you don't plan on using CloudFront and decide to just host content from S3 only.

```scss
---
AWSTemplateFormatVersion: '2010-09-09'


Description: Simple S3 Bucket to host static public website.


Resources:


  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: FIXME.com
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        BlockPublicPolicy: false
        IgnorePublicAcls: false
        RestrictPublicBuckets: false
      Tags:
        -
          Key: Description
          Value: FIXME
        - Key: Project
          Value: FIXME.com
      VersioningConfiguration:
        Status: Enabled
      WebsiteConfiguration:
        ErrorDocument: 404.html
        IndexDocument: index.html
          
  BucketPolicyDataSync:  
    Type: 'AWS::S3::BucketPolicy'  
    Properties:  
      Bucket:  !Ref S3Bucket
      PolicyDocument:
        Statement:  
        -  
          Sid: "AllowAccesToIAMRole"  
          Action:  
            - "s3:GetObject"
            
          Effect: "Allow"  
          Resource:  
            Fn::Join:  
              - ""  
              -  
                - "arn:aws:s3:::"  
                -  
                  Ref: "S3Bucket"  
                - "/*"  
          Principal: "*"            

  WWWBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: www.FIXME.com 
      AccessControl: BucketOwnerFullControl
      WebsiteConfiguration:
        RedirectAllRequestsTo:
          HostName: FIXME.com

Outputs:


  S3BucketName:
    Value: !Ref S3Bucket
    Description: S3 Bucket for object storage


  S3BucketARN:
    Value: !GetAtt S3Bucket.Arn
    Description: S3 bucket ARN
```

1. Navigate to the region closest to you and go to the CloudFormation service
2. Upload your updated CFT stack to create your new S3 buckets for hosting your static website files.
3. Capture your S3 Endpoint URL - you can find this by navigating to your new S3 Bucket > Properties > Static web site hosting > Endpoint

#### Create CloudFront Distribution

1. Navigate to the AWS Cloudfront service and select Create distribution > Web > Get started
2. Under Origin Domain name, paste your S3 Endpoint and remove the "https://" from the front. Leave remaining Origin settings as default.
3. Under Distribution Settings, Alternate Domain Names (CNAMEs) enter your A records created in the Route 53 public hosted zone section above and enter index.html as Default Root Object, for example:
4. Select AWS Certificate Manager > register SSL cert on your new domain name this can take 30+ minutes to register and must be created in us-east-1 only in order to use on CloudFront.
5. Copy your new Cloudfront address, it will look like this: FIXME.cloudfront.net

#### Create IAM Policy and IAM User for Github Actions
##### IAM Policy
1. Navigate to IAM and [create a new IAM Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html#access_policies_create-json-editor) using the JSON editor.
2. Paste the following policy contents into the JSON editor and update all "FIXME" values for your bucket name, AWS account ID and CloudFront ID which is the alphanumeric 14 characters of your associated Cloudfront distribution:

```scss
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Resource": [
                    "arn:aws:s3:::FIXME-bucket-name",
                    "arn:aws:s3:::FIXME-bucket-name/*"
                ],
                "Sid": "VisualEditor1",
                "Effect": "Allow",
                "Action": [
                    "s3:*"
                ]
            },
            {
                "Sid": "VisualEditor2",
                "Effect": "Allow",
                "Action": "cloudfront:*",
                "Resource": "arn:aws:cloudfront::FIXME-aws-account-number:distribution/FIXME-distribution-id"
            }
        ]
    }
```
##### IAM User
1. Create a new IAM User with programmatic access and attach the IAM Policy you just created above.
2. Copy the AWS access key ID and Secret access key to a safe location such as secrets manager or key vault, as they'll be used in the GitHub Action Workflow setup below.

#### Create GitHub Action Workflow 

1. From within your GitHub repo navigate to .github/workflows/ and create a new file called `build-and-deploy.yml`
2. Copy and paste the following into your newly created GitHub Action workflow and update the FIXME value for the region that your S3 bucked was deployed in.

```scss
name: CI / CD

# Controls when the action will run. 
on:
  # Triggers the workflow on push for the main branch
  push:
    branches: [ main ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
  
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: 'FIXME'

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: 2.7
        bundler-cache: true
    - uses: actions/cache@v2
      with:
        path: vendor/bundle
        key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
        restore-keys: |
          ${{ runner.os }}-gems-
    - name: Install dependencies
      run: |
        gem install bundler
        gem install jekyll
        bundle config path vendor/bundle
        bundle install --jobs 4 --retry 3        
    - name: "Build Site"
      run: bundle exec jekyll build
      env:
        JEKYLL_PAT: ${{ secrets.JEKYLL_PAT }}
    - name: "Deploy to AWS S3"
      run: aws s3 sync ./_site/ s3://${{ secrets.AWS_S3_BUCKET_NAME }} --acl public-read --delete --cache-control max-age=604800
    - name: "Create AWS Cloudfront Invalidation"
      run: aws cloudfront create-invalidation --distribution-id ${{ secrets.AWS_CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```
#### Create GitHub Action Secrets
1. Navigate to Your Repo > Settings > Secrets > Actions
2. Configure The GitHub Action Secrets for the following:
- `AWS_ACCESS_KEY_ID` - The AWS access key ID associated with the programmatic IAM User created previously.
- `AWS_SECRET_ACCESS_KEY` - The AWS secret key ID associated with the programmatic IAM User created previously.
- `AWS_S3_BUCKET_NAME` - Your AWS bucket name hosting your website, for example: jennasprattler.com
- `AWS_CLOUDFRONT_DISTRIBUTION_ID` - The alphanumeric 14 characters of your associated Cloudfront distribution fronting your S3 bucket's website.

#### Deploy your Jekyll website using GitHub Actions
1. 