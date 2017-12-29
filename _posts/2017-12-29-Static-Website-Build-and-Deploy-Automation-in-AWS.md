---
layout: post
title: Static Website Build and Deployment Automation in AWS
---

Since the migration to [AWS](https://aws.amazon.com/), [Troll Purse](http://trollpurse.com) has been hard at work to automate developer processes. Today, you will read about examples on how to automate the build of a static website using [node](https://nodejs.org), [gulp](https://gulpjs.org), [AWS CodeBuild](https://aws.amazon.com/codebuild/), [S3](https://aws.amazon.com/s3/) static websites, and [Lambda](https://aws.amazon.com/lambda/).

# The Problem to Solve

Troll Purse is a fan of automation. Continuous Integration and Continuous Delivery of Troll Purse games and services are an end goal at Troll Purse. Previously, while hosting on [Digital Ocean](https://www.digitalocean.com/), Troll Purse development was an overly involved process. Now, Troll Purse developers simply need to update and push website changes to have them automatically deployed to AWS.

## The Old Process

Here is a list of what was done to develop and deploy updates to the [website](http://trollpurse.com) in Digital Ocean.

1. Develop changes and push to repository.
2. Find the person with the [SSH](https://en.wikipedia.org/wiki/Secure_Shell) credentials to SSH to the single server in Digital Ocean.
3. Give a list of `git` commands to pull the latest code.
4. Give a list of build commands for `gulp` to build the latest website content.
5. Copy the built data to the website directory served by [nginx](https://www.nginx.com/).
6. Flush any caching the server chose to do.
7. Verify website changes took place.

## What was Desired

Right, not a very automated approach when peeking behind the curtian. Yes, developers at Troll Purse eventually automated steps 3 - 5 using [bash scripts](https://en.wikipedia.org/wiki/Bash_(Unix_shell)). However, the triggering of a website build on a push to our production branch was desired. Developers would then never need to leave the context of thier local development environment to see changes on the servers. This helps create [abstracted developers](https://www.joelonsoftware.com/2006/04/11/the-development-abstraction-layer-2/) who would rather be developing, as opposed to running operations, anyways.

The biggest desire of Troll Purse developers was to execute steps 1 and 7 only.

# The .sln

Aside from the clever pun above, the Troll Purse team spent a lot of time thinking of how to automate the AWS hosted static website build and deployment. 

## Solution Iteration 1

The team first created a simple three line script that could be executed from a developers box to deploy to production:

```
npm install
gulp build
aws s3 cp ./built s3://our-bucket-name-for-prod-website/ --recursive
```

This was fine and dandy - for a while. There were several issues with this practice. The issues are listed below.

- Attempts to make this a script failed due to the nature of credentials gathering in the [aws-cli](https://aws.amazon.com/cli/).
- Developers would each need thier own IAM role and secret keys - a management nightmare for security (there are solutions to this but not in our scope)
- Still pretty manual - the desire was to have developers push and be done with thier updates.

## Solution Iteration 2

Using what was already at hand, developers at Troll Purse decided to integrate [CodeBuild](https://aws.amazon.com/codebuild/) with [Github](https://www.github.com) to automate builds. The original plan was to have CodeBuild write the artifacts directly to the public website's bucket. The configuration was easy to do so, in fact, here it is:

```yml
version: 0.2

phases:
  install:
    commands:
      - npm install --global gulp-cli
  pre_build:
    commands:
      - npm install
  build:
    commands:
      - npm install
      - gulp build
artifacts:
  files:
    - ./**/*
  discard-paths: no
  base-directory: built
```

This configuration runs on an Ubuntu build server using the NodeJS configuration provided by Amazon.

Unfortunately, due to CodeBuild logic, this would not work. CodeBuild insisted on writing to the bucket with a specific prefix to the keys. Considering the purspose of CodeBuild, this made sense. Therefore, the team did not complain, but moved on.

So, the team looked at building with CodeBuild and deploying with [CodeDeploy](https://aws.amazon.com/codedeploy/) - which seemed logical. This also presented a road block. CodeDeploy will only deploy to Docker Containers or EC2 instances.

With CodeBuild all setup and running properly, the team came up with a simple solution to automate website deployment from the CodeBuild build artifacts.

## Solution Iteration 3

The Troll Purse team decided to use [AWS Lambda](https://aws.amazon.com/lambda/) as a deployment tool for static websites hosted in S3. Honestly, this was extremely easy and required very little code or configuration. Amazon provides plenty of templates in the management console for Lambda, as well a great plugin for Visual Studion 2015/2017 (the lambda function was written in C#) that also provides templates.

The template used was the **Simple S3 Function** template. From there, the code was modified as follows (yes you get to see it all!):

```csharp
using Amazon.Lambda.Core;
using Amazon.Lambda.S3Events;
using Amazon.S3;
using Amazon.S3.Model;
using System;
using System.Threading.Tasks;

// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.Json.JsonSerializer))]

namespace TrollLambda
{
    public class Function
    {
        private readonly IAmazonS3 S3Client;
        private readonly string TargetBucket;
        private readonly string StripPrefix;

        public Function()
        {
            S3Client = new AmazonS3Client();
            TargetBucket = "trollpurse.com"; //Static website hosting is put in buckets named after domain.
            StripPrefix = "trollpurse-website/"; //Code build prepends a key prefix that we do not desire.
        }

        public Function(IAmazonS3 s3Client, string targetBucket, string stripPrefix = null)
        {
            TargetBucket = targetBucket;
            S3Client = s3Client;
            StripPrefix = stripPrefix;
        }

        public async Task<string> CopyOnS3Put(S3Event evnt, ILambdaContext context)
        {
            var s3Event = evnt.Records?[0].S3;
            if (s3Event == null)
            {
                context?.Logger.LogLine($"No Records in Event");
                return null;
            }

            try
            {
                CopyObjectRequest request = new CopyObjectRequest
                {
                    SourceBucket = s3Event.Bucket.Name,
                    SourceKey = s3Event.Object.Key,
                    DestinationBucket = TargetBucket,
                    DestinationKey = s3Event.Object.Key.StartsWith(StripPrefix)
                        ? s3Event.Object.Key.Remove(0, StripPrefix.Length)
                        : s3Event.Object.Key
                };
                CopyObjectResponse response = await S3Client.CopyObjectAsync(request);
                return response.HttpStatusCode.ToString();
            }
            catch (Exception e)
            {
                context?.Logger.LogLine($"Error getting object {s3Event.Object.Key} from bucket {s3Event.Bucket.Name}. Make sure they exist and your bucket is in the same region as this function.");
                context?.Logger.LogLine(e.Message);
                context?.Logger.LogLine(e.StackTrace);
                throw;
            }
        }
    }
}
```

Notice that there are no AWS secret keys used here, using roles is an excellent way to develop secure applications without copious amounts of key sharing.

The function was confiured in the management console to respond to S3 PUT, Multipart Upload Complete, and POST Object creation in the source bucket (where CodeBuild pushes artifacts) with a filter on the prefix CodeBuild appends for a project build. It required write access to the `TargetBucket` (the website public S3 bucket), read access to the `SourceBucket` (the CodeBuild output bucket), and permissions for log streams in [AWS CloudWatch](https://aws.amazon.com/cloudwatch/). Most of that can be auto-configured in the management section for Lambda functions.

# Other Solutions Discussed

The entire process of prototyping, discussing, developing, and implementing the automation of the Troll Purse website development took about eight hours. During that time some other options were mentioned that will be covered.

## Github and Jekyll

Yes, the Troll Purse website source is [hosted on github](https://www.github.com/trollpurse/website). As some may know, github offers a product called [Github Pages](https://pages.github.com/). This product allows developers to leverage [Jekyll](https://jekyllrb.com/) to produce static websites. Our website is static - so it seemed like a natrually choice, at first.

There is a reason Troll Purse did not use github pages, as it does for the [Troll Purse blog](http://blog.trollpurse.com), to build and host the [Troll Purse website](http://trollpurse.com). This reason was expansion options and future development.

In contrast to the blog, the Troll Purse website is not close to a final iteration. The product is still developing. There are plans that may include moving the site away from being a static entity to a dynamic website. With the current build process, Troll Purse will only need to change 25% of the automoation implementation. This covers the reasons of offering options for expansion and flexibility in future development.

## Third Party Provider for Static Website Hosting

The reason Troll Purse did not use a third party provider for hosting the website is manifold. Including the reasons mentioned for Github and Jekyll, the other reasons are as follows. Cost and environment.

Most of the other providers for hosting static websites cost around $5 / month to just serve content. Troll Purse pays only 10% of that cost to do so in AWS with only minor hassles in automation development.

Other than cost, environment played a big role. Honestly, most static website hosting platforms still use [confusingly bad interfaces](https://documentation.cpanel.net/display/68Docs/The+cPanel+Interface), products easily [vulnerable to malicious entities](https://krebsonsecurity.com/2013/04/brute-force-attacks-build-wordpress-botnet/), and allow black-hat [developers](https://www.godaddy.com/help/malicious-wordpress-plugins-12035) to create products. All in all, it seemed safer we use a single entity providing a good backend and create a minimal hosting environment.

# Results

After only eight hours of work, our web developers can now make updates to the website, push changes, and not worry about development operations. This is a big win for Troll Purse. Hopefully, this guides you with your own plans of web hosting in AWS.