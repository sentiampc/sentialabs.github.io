---
layout: post
title: Serverless Screenshot service with Lambda
author: Sander van de Graaf
email: sander.van.de.graaf@sentia.com
---

Recently a client requested a feature which involved screenshots of random urls. Now, there are several services out there which will do this for you.
Most of these services have interesting rest api's and pricing models. I really wanted to develop something with Serverless, and took this as an opportunity to check things out. This will run on the Amazon services (eg: Lambda).

You can find all the source code mentioned in [this repository](https://github.com/svdgraaf/serverless-screenshot).

Quick installation
==================
If you just want to launch the service yourself, you can use this magic button which will setup everything for you in your AWS account through the magic of CloudFormation:

[![Launch Awesomeness](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=serverless-screenshot-service&templateURL=https://s3-eu-west-1.amazonaws.com/serverless-screenshots-service/2016-09-23T12%3A50%3A03/template.yml)

Overview
--------
We're going to create a service to which we can `POST` a url, it will capture and store a screenshot of the given url for us, and return a url where we can download the screenshot.
Also, we want to have different thumbnail sizes of every screenshot, and we want to be able to list all the available thumbnail sizes of a url.

The architecture of what we're going to build, looks something like this:

![Lambda Screenshot Architecture](https://github.com/svdgraaf/serverless-screenshot/raw/master/docs/architecture.png)

The app posts to Api Gateway, which will trigger a Lambda function, which will take the screenshot, and upload it to the bucket. It will then return the url for the created screenshot.

When the file gets uploads (putObject) to the S3 bucket, it will trigger the `Create Thumbnails` function, which will take the screenshot as input, and use ImageMagick to create several thumbnail versions of the given image, and upload those to the same bucket as well.

Lastly, there will be a function which the app can call, to get a list of all available thumbnail sizes.

The App can use the returned urls to present them to the enduser, and their browser will download it through CloudFront from S3.

You can find all the source code for this project in this [repository](https://github.com/svdgraaf/serverless-screenshot).

Getting Started
===============
This is my first project in NodeJS, so bare with me (I'm more of a Python guy), any tips or PR's are greatly appreciated 😄

When you download the project, you can just run a `npm install`, to get all the requirements installed, and get going. If you want to follow along from scratch, read on. Be sure to install the latest Serverless version (`npm install -g serverless`).


Overview
--------
For this project, we're going to setup:

 * One Lambda function, triggered by a `POST` event
 * One Lambda function, triggered by a `GET` event
 * One Lambda function, triggered by an S3 `CreateObject` event
 * Extra IAM Role access for the screenshot bucket
 * An S3 bucket for the screenshots
 * A CloudFront distribution for serving the screenshots

Let's get started
-----------------
We start the project by creating a new project (in the current directory):

```bash
serverless create --template aws-nodejs
```

We now have a directory with a couple of files:

```bash
ls -l
total 32
-rw-r--r--  1 svdgraaf  staff    54 Sep  6 19:44 README.md
-rw-r--r--  1 svdgraaf  staff    63 Sep  6 19:48 event.json
-rw-r--r--  1 svdgraaf  staff   128 Sep  6 19:48 handler.js
-rw-r--r--  1 svdgraaf  staff  1754 Sep  6 19:48 serverless.yml
```

The `handler.js` file will contain our functions, and with `event.json` we can simulate our calls. I later moved this to an `events` directory, so I could simulate multiple events for the other calls.

Let's first take a look at the `serverless.yml` file, which contains the configuration for our Service.

Provider configuration
----------------------
First, we need to setup our provider configuration, in this case, we'll use `aws`, with `nodejs4.3`. We also add an api key, so we can securely use our api, and add some iam rules to allow the functions to list and upload to the S3 bucket.

```yaml
provider:
  name: aws
  runtime: nodejs4.3
  stage: dev

  # We want to lock down the ApiGateway, so we can control who can use the api
  apiKeys:
    - thumbnail-api-key

  # We need to give the lambda functions access to list and write to our bucket, it needs:
  # - to be able to 'list' the bucket
  # - to be able to upload a file (PutObject)
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - "s3:ListBucket"
        - "s3:PutObject"
      Resource: { "Fn::Join" : ["", ["arn:aws:s3:::", { "Ref" : "ThumbnailBucket" } ] ]  }
```

Functions
---------
We need to create two functions. One for taking a screenshot (called, you guessed it: `takeScreenshot`), one for creating a thumbnail of the screenshots (`createThumbnails`), and one for listing the available screenshot sizes (`listScreenshots`).


```yaml
functions:
  takeScreenshot:
    handler: handler.take_screenshot
    timeout: 15  # our screenshots can take a while sometimes
    events:
      - http:
          path: screenshots
          method: post
          # Marking the function as private will require a valid api-key
          private: true
          request:
            # we want to mark the url param as required
            parameters:
              querystrings:
                url: true

  listScreenshots:
    handler: handler.list_screenshots
    timeout: 15
    events:
      - http:
          path: screenshots
          method: get
          private: true
          request:
            parameters:
              querystrings:
                url: true

  createThumbnails:
    handler: handler.create_thumbnails
    events:
      # this event type will also create the screenshots bucket and trigger
      # the lambda function every time a file is uploaded (ObjectCreated)
      - s3:
          bucket: ${self:custom.bucket_name}
          event: s3:ObjectCreated:*
```

Custom Resources
----------------
We also need a CloudFront distribution to serve our screenshots. This is not supported by Serverless out of the box, so we need to create a custom resource for it. We also define some `Output`'s, so we can use these later on.

```yaml
resources:
  Outputs:
    ScreenshotBucket:
      Description: "Screenshot bucket name"
      Value: ${self:custom.bucket_name}
    CloudFrontUrl:
      Description: "CloudFront url"
      Value: {"Fn::GetAtt": "CloudFrontEndpoint.DomainName"}

  Resources:
    # Create an endpoint for the S3 bucket in CloudFront
    # this configuration basically just sets up a forwarding of requests
    # to S3, and forces all traffic to https
    CloudFrontEndpoint:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Enabled: True
          DefaultCacheBehavior:
            TargetOriginId: ScreenshotBucketOrigin
            ViewerProtocolPolicy: redirect-to-https
            ForwardedValues:
              QueryString: True
          Origins:
            -
              Id: ScreenshotBucketOrigin
              DomainName: ${self:custom.bucket_name}.s3.amazonaws.com
              CustomOriginConfig:
                OriginProtocolPolicy: http-only
```

Plugin variables
----------------
We use the [stage-variables](https://www.npmjs.com/package/serverless-plugin-stage-variables) plugin, so we can set stage variables in ApiGateway, which we can then use in our Lambda functions. You can set those in the `custom` section. This also allows us to use CloudFormation references inside our Lambda functions. CloudFormation will fill in the values with the actual output for us.

```yaml
custom:
  # change this, so it's unique for your setup
  bucket_name: ${self:provider.stage,opt:stage}-${env:USER}-screenshots

  # these variables are passed on through ApiGateway, to be used
  # in your functions in the event.stageVariables
  stageVariables:
    bucketName: ${self:custom.bucket_name}
    endpoint: {"Fn::Join": ["", ["https://", { "Fn::GetAtt": "CloudFrontEndpoint.DomainName" }, "/"]]}
```

Defining functions
==================
In the `handler.js` file, you should define your functions. In our case, it will hold three functions: `take_screenshot`, `list_screenshots` and `create_thumbnails`. You can use the `handler` definition in in the `functions` section of `serverless.yml` to optionally use different files if you want to separate them.
The functions will always receive three variables: `event`, `context` and `cb`. The `event` will contain the event which triggered the Lambda (in our case, either an ApiGateway call, or the S3 bucket event). The `context` variable will contain the Lambda context you can use to figure out how much memory you have, the platform, etc. The `cb` (callback) you can use to signal for errors or success.

Event variables
---------------
Depending from which service you receive an event, you will receive different events. In the case of our Api Gateway events, we get a lot of stuff, headers, query parameters, stage variables, etc. In our case, we're only using `event.query` and `event.stageVariables`, but there is also `event.headers`, `event.body`, etc. The mapping to the event is defined by Serverless, and the default mapping (and fields) can be seen in the [repository](https://github.com/serverless/serverless/blob/master/lib/plugins/aws/deploy/compile/events/apiGateway/lib/methods.js#L98).

Extra binaries
--------------
For this project, we will need some extra binaries ([PhantomJS](http://phantomjs.org/) in particular), which take the screenshots. We also use [ImageMagick](http://www.imagemagick.org/script/index.php), but that is provided by AWS by default in the Lambda image, so we don't package that separately.
Serverless will package any extra files in the project directory automatically. So adding extra binaries is as simple as just creating a directory, and adding the files.

If you need (compiled) binaries, you can use the Amazon Lambda AMI, and compile [your own binaries](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html). The [project repository](https://github.com/svdgraaf/serverless-screenshot) already contains the binaries you can use. You can also use this [docker container](https://hub.docker.com/r/lambci/lambda/) to compile binaries in.

Deploying
=========
Now that we have everything in place, we can deploy the application. It's a good idea to use different deploy environments, so let's start with a `dev` environment: `sls deploy -s dev`. This will zip everything together (including the binaries), create a deployment bucket, upload the zipfile, and update the cloudformation template with all the resources that we need.

```python
sls deploy -s dev --verbose
```

(the `--verbose` will show you all the CloudFormation output during deployment, very nice!).

After the deployment is done, you will see the end results:

```
Service Information
service: lambda-screenshots
stage: dev
region: us-east-1
endpoints:
  POST - https://123g1jc802.execute-api.us-east-1.amazonaws.com/dev/screenshots
  GET - https://123g1jc802.execute-api.us-east-1.amazonaws.com/dev/screenshots
functions:
  lambda-screenshots-dev-listScreenshots: arn:aws:lambda:us-east-1:123123123123:function:lambda-screenshots-dev-listScreenshots
  lambda-screenshots-dev-createThumbnails: arn:aws:lambda:us-east-1:123123123123:function:lambda-screenshots-dev-createThumbnails
  lambda-screenshots-dev-takeScreenshot: arn:aws:lambda:us-east-1:123123123123:function:lambda-screenshots-dev-takeScreenshot
```

![Awesome](http://i.giphy.com/5GoVLqeAOo6PK.gif)

**Great**, everything is up and running!

Now we can test our functions. You can use your favorite http client like `curl` or `postman`, etc.

You can find your apikey's in the AWS ApiGateway console. You need to set the `x-api-key` header with the api key for every request.

Creating a screenshot
---------------------
So to create a screenshot for `google.com`:

```bash
$ curl -X POST https://123g1jc802.execute-api.us-east-1.amazonaws.com/dev/screenshots?url=http://google.com/ -H "x-api-key: 123123Qqws6SFh1t123123vsXpo5VfFI5MNJf123"
{
	"hash": "6ab016b2dad7ba49a992ba0213a91cf8",
	"key": "6ab016b2dad7ba49a992ba0213a91cf8/original.png",
	"bucket": "dev-foobar-screenshots",
	"url": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/original.png"
}
```

Listing all screenshot sizes
----------------------------
And to get all the different available sizes for `google.com`:

```bash
$ curl -X GET https://123g1jc802.execute-api.us-east-1.amazonaws.com/dev/screenshots?url=http://google.com/ -H "x-api-key: 123123Qqws6SFh1t123123vsXpo5VfFI5MNJf123"
{
	"100": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/100.png",
	"200": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/200.png",
	"320": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/320.png",
	"400": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/400.png",
	"640": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/640.png",
	"800": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/800.png",
	"1024": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/1024.png",
	"1024x768": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/1024x768.png",
	"320x240": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/320x240.png",
	"640x480": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/640x480.png",
	"800x600": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/800x600.png",
	"original": "https://foobar.cloudfront.net/6ab016b2dad7ba49a992ba0213a91cf8/original.png"
}
```

Pricing
=======
So what will this cost you? If every call would take around 10 seconds (which is a safe bet, usually they are way faster). You could make 250.000 calls per month in the free tier, which wouldn't cost you a thing.

So creating 100.000 screenshots **per month**, with 15 thumbnails per screenshot would be absolutely free.

Conclusion
==========
Taking screenshots is something which is perfectly fine to do with Lambda. There is no need to setup queuing, batch workers, etc. Lambda can do the screenshotting, thumbnailing and storage. Whenever a request comes in, it will automatically spin up, solving autoscaling out of the box for you. Awesome!

[![Launch Awesomeness](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=serverless-screenshot-service&templateURL=https://s3-eu-west-1.amazonaws.com/serverless-screenshots-service/2016-09-23T12%3A50%3A03/template.yml)
