# Introduction
Static website for GoodTrip Ink Tattoo and Gallery using [Slides 3.1.2](https://designmodo.com/slides/) framework.

[ ![Codeship Status for dayelostraco/goodtripink](https://app.codeship.com/projects/2995e710-6833-0135-a28b-5ec5668067cc/status?branch=master)](https://app.codeship.com/projects/241240)

Both branches have their own Domains, S3 Hosted Website buckets, CloudFront distributions and Codeship automated deployments. However, both branches share a single Wildcard SSL Certificate that is managed in AWS Certificate Manager (id: `*.goodtrip.ink`).

## Static File Structure

```
/
├─ index.html
├─ error.html
├─ css/
│  ├─ slides.css
│  └─ custom.css (For custom CSS)
├─ js/
│  ├─ slides.js
│  ├─ plugins.js
│  └─ custom.js (For custom JS)
├─ scss/*
└─ assets/
   ├─ ai/*
   ├─ img/*
   ├─ psd/*
   ├─ resume/*
   └─ svg/*
```

## S3 Configuration

### `sanfrancisco`

The static website is hosted via the S3 Bucket `goodtrip.ink` with full anonymous GET access. To reduce URL length, I have also created the `www.goodtrip.ink` bucket which simply redirects requests to the base `goodtrip.ink` bucket. 

In addition, I have created S3 buckets for `www.meetdayel.com` and `meetdayel.com` URLs that also serve as redirects to the S3 bucket hosted website for `goodtrip.ink`.

### `master`

The static website is hosted via the S3 Bucket `goodtrip.ink` with full anonymous GET access. To reduce URL length, I have also created the `www.goodtrip.ink` bucket which simply redirects requests to the base `goodtrip.ink` bucket. 


## CloudFront

The CloudFront distribution simply caches the contents of the `goodtrip.ink` bucket across all Edge Nodes using a wildcard SSL certificate (maintained in AWS Certificate Manager) that contains the following URLs:

* goodtrip.ink
* *.goodtrip.ink

Furthermore, CloudFront redirects all HTTP requests to the HTTPS protocol and GZips all requested content.

For each distribution, the CNAMEs are configured for the URLs that will be routed to the content cache.

## Route 53 Configuration

The Route 53 configuration for both versions of the statically hosted site is straight forward.

### `master`
* A Record for goodtrip.ink to CloudFront distribution with corresponding CNAME
* A Record for www.goodtrip.ink to CloudFront distribution with corresponding CNAME

## Codeship Configuration
There is a Codeship Deployment Pipeline for all branches that are to be published to an S3 hosted website. Currently, the `master` branch is supported and live.

You'll need to configure the following:
* Create a new Deployment Pipeline with the Git branch name you want used.
* Select the `Custom Script` option.
* Add the script located in the Deployment Section of this README.
* Edit the script to point to the s3 bucket you want cleared out and for the website files uploaded to. This needs to be done in two places within the script.
* You will need to create an AWS IAM user that has GET, PUT, DELETE policy access to the S3 buckets used for website hosting.
* Add Environment Variables to the Codeship project. You will need to set up variables for AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION, CLOUDFRONT_SANFRANCISCO_ID and CLOUDFRONT_CHARLESTON_ID.

NOTE: If you want to tweak the Cache-Control headers, you an update the EXP_DATE variable in the script.

## Deployment

Automated CodeShip deployments to the S3 hosted website on all commits to the `sanfrancisco` and `charleston` branches.

CodeShip deployment uses the Development pipeline to run AWS CLI commands

* Clones the latest branch
* Deletes all files contained within the S3 website bucket
* Uploads all all branch files sans .git, .gitignore and README.md
* Sets read access on all files to Public-Read
* Sets the Cache Control header on all files to 30 days out
* Sets AWS CLI mode to allow Preview level features (needed for CloudFront cache invalidation commands)
* Invalidates the CloudFront cache so all new changes are available and re-cached

All that is needed for deployments is for you to commit/push changes to a branch that has Codeship Development pipelines configured and you're done!

The Custom Codeship script that is currently in Production is:
```
# Install AWS CLI
pip install awscli

# Codeship clones your project into this directory
cd ~/clone

# Set Expiration Date Session Var to +1 month of current system time
EXP_DATE=$(date --date="+1 week" "+'%Y-%m-%dT%H:%M:%SZ'")

# Delete existing S3 Files
aws s3 rm s3://goodtrip.ink --recursive

# Copy cloned files from GitHub to S3 Bucket
# Set each file to Public-Read, expires at the EXP_DATE var and Cache-Control header to 30 days max age
aws s3 mv ~/clone s3://goodtrip.ink --exclude '.git/*' --exclude '.gitignore' --exclude 'README.md' --acl public-read --expires $EXP_DATE --cache-control max-age=604800 --recursive

# Invalidate the CloudFront cache via preview level features
aws configure set preview.cloudfront true
aws cloudfront create-invalidation --distribution-id $CLOUDFRONT_ID --paths index.html error.html assets/img/* assets/svg/* /index.html /error.html /assets/img/* /assets/svg/*
````
