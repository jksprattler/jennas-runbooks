# Jekyll Site Hosted on AWS S3 using Github Actions

_Last updated: July 01, 2021_

## Overview
The purpose of this runbook is to define the necessary steps in order to deploy a secure static website hosted on an AWS S3 Bucket, served by an AWS CloudFront Distribution (CDN) which will all be automed using a GitHub Action Workflow.

Our use case will be the following website: [jennasprattler.com](https://jennasprattler.com/) or [www.jennasprattler.com](https://www.jennasprattler.com/)

### Pre-requisites
- Static website 
  - I'll be using the Jekyll theme found here: [modern-resume-theme](https://github.com/sproogen/modern-resume-theme)
- GitHub Account and Repo to store your website files
- AWS Account

### High-level Overview - Traffic Flow
![jekyll-web-flow](/images/jekyll-web-flow.jpg)
- 

### High-level Overview - CI/CD Flow
![jekyll-code-deploy](/images/jekyll-code-deploy.jpg)