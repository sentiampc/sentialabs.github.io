---
layout: post
title: CloudFormation SSM Secure String (using an inline boto3 custom resource)
---

Currently, CloudFormation doesn't have support for the Parameter Store Secure Strings, which is unfortunate. This is just a matter of time though, as AWS will probably announce support at some point in the future.

Fortunately there is a "nice" workaround, called [Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html). This works by creating a Lambda function, which creates whatever you want to create.
This opens a whole can of worms though, you end up writing a lambda function, uploading it to s3, calling it from within CloudFormation, etc.
And by doing so, now you have to maintain the file in S3, take care of the packaging, versioning and deploy process, while you JUST WANT THE D*MN THING TO WORK.

There is a somewhat nice solution for this though. Using a special notation, you can actually embed the source code for a Lambda function within your template.

Using the `ZipFile` type for the `Code` parameter, you can write (messy) inline functions. There is a limit (4000 bytes), but that's more than enough to run something decent. Here's an example:

```yml
MyFooCustomResourceFunction:
  Type: AWS::Lambda::Function
  Properties:
    Handler: "index.handler"
    Runtime: python3.6
    Timeout: 30
    Role: !GetAtt MyRole.Arn
    Code:
      ZipFile:
        "Fn::Join":
          - "\n"
          -
            - "import boto3"
            - "import json"
            - "from botocore.vendored import requests"
            - ""
            - "def handler(event, context):"
            - "    print 'woohaa!'"
```

And because `boto3` and `requests` are available by default in the Python runtime, you don't actually have to do any packaging, yay!

Let's take this a step further.

You can actually write a lambda function which calls a `boto3` function using the parameters from the custom resources and output the `boto3` response as JSON. Which you can then turn into attributes for the resource, which can then be used using the `!GetAtt` function for other resources.

Wait wut?
---------

A `Custom Resource` is just like any other resource, with parameters and all. Whenever you create a Custom Resource, during the Create (and Update) state of the Custom Resource you are obligated to respond to CloudFormation with a [specific response](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-responses.html). In the call that you're sending to CloudFormation, you give it a JSON response with the actual status of the Custom Resource (Failed, Created, Deleted, Updated).

This JSON response can also have a list of arbitrary key/values, which will be exposed as `attributes` for the Custom Resource inside your CloudFormation template. These attributes can be used as any other attribute in CloudFormation, using the `!GetAtt` function.

This is the python function which does all this magic for you:

```python
import boto3
import json
import ast
from botocore.vendored import requests

def flatten(d, parent_key=''):
    items = []
    for k, v in d.items():
        try:
            items.extend(flatten(v, '%s%s.' % (parent_key, k)).items())
        except AttributeError:
            items.append(('%s%s' % (parent_key, k), v))
    return dict(items)

def handler(event, context):
    props = event['ResourceProperties']
    data = {}
    client=boto3.client(props['Service'])
    event_type = event['RequestType']

    # we default to success if no event_type is defined
    status = 'SUCCESS'
    reason = ''
    data = {}

    # Check if the current event type is configured, if so, run it
    if event_type in ['Create', 'Delete', 'Update'] and event_type in props:
        method=getattr(client, props[event_type]['Method'])
        kwargs = {}

        # We get text values from CloudFormation properties, this won't
        # work for booleans and numbers (boto is pretty anal about this).
        # so we convert them using the AST library to proper types
        for key, value in props[event_type]['KwArgs'].items():
            try:
                kwargs[key] = ast.literal_eval(value)
            except Exception as e:
                print(str(e))
                kwargs[key] = value

        # Try to call the actual method in boto
        try:
            data = method(**kwargs)
            status = 'SUCCESS'
            reason = ''
        except Exception as e:
            status = 'FAILED'
            reason = str(e)
            data = {}

    response_data = {
        'Status': status,
        'Reason': reason,
        'PhysicalResourceId': event['LogicalResourceId'],
        'StackId': event['StackId'],
        'RequestId': event['RequestId'],
        'LogicalResourceId': event['LogicalResourceId'],

        # We need to flatten the response data object, so we can also access
        # nested objects and values, because in CF only the first level
        # of properties will become available as attributes.
        # So {"foo": {"bar": "baz"}} becomes {"foo.bar": "baz"}, and can be
        # used as !GetAtt myresource.foo.bar, as expected
        'Data': flatten(data)
    }

    # print the callback for debugging
    print(response_data)

    resp = requests.put(event['ResponseURL'], data=json.dumps(response_data))
    if resp.status_code != 200:
        print(resp.text)
        raise Exception(resp.text)
    return
```
With this you can create a custom resource, for example like so:

```yaml
MyCustomResource:
  Type: Custom::Foobar
  Properties:
    Service: S3
    Create:
      Method: put_object
      KwArgs:
        Body: "foobar"
        Bucket: "my-bucket"
        Key: 'my-file.txt'
```

The full usage description:

```yaml
MyCustomResource:
  Type: Custom::Foobar
  Properties:
    # service name for client, check the boto documentation for the right name
    # to use: http://boto3.readthedocs.io/en/latest/reference/services/index.html
    Service: S3
    # There are 3 different "main" properties:
    # - Create (when creating the resource)
    # - Update (when updating the resource)
    # - Delete (when deleting the resource)
    Create:
      # the method use on the boto client
      Method: put_object
      # the arguments which need to be passed to the method
      # see the boto documentation, eg:
      # http://boto3.readthedocs.io/en/latest/reference/services/s3.html#S3.Client.put_object
      KwArgs:
        Body: "foobar"
        Bucket: "my-bucket"
        Key: 'my-file.txt'
    # This is the Delete command, if needed, optional
    # Delete:
    #   Method: delete_object
    #   KwArgs:
    #     Bucket: !Ref MyS3Bucket
    #     Key: 'myPasswordFile.txt'
    # This is the Update command, if needed, optional
    # Update:
    #   Method: put_object
    #   KwArgs:
    #     Body: "foobar"
    #     Bucket: "my-bucket"
    #     Key: 'my-file.txt'

    ServiceToken:
      "Fn::GetAtt":
        - BotoCustomResource
        - Arn
```

Full example
------------

So how this does work? Well, let's do this: as an example, let's create a CloudFormation template, which:

  - Creates an s3 Bucket
  - Fetches a SECRET parameter value from the parameter store
  - Writes the secret to a file on the newly created S3 bucket

Of course, using the Secret in this way is discouraged, but this shows you that you can create the custom resource function once, and re-use it multiple times in the same template.

*important*
Be sure to give the custom resource role the correct permissions to call the actual boto function, or things will fail in CloudFormation. In the template below, notice the lines within the `LambdaExecutionRole` resource:

```yaml
- Effect: Allow
  Action:
    - s3:PutObject
    - s3:DeleteObject
    - ssm:GetParameter
  Resource: "*"
```

Here's the template which does everything:

```yml
---
  AWSTemplateFormatVersion: '2010-09-09'
  Description: Foobar
  Resources:

    # create an S3 bucket
    MyS3Bucket:
      Type: "AWS::S3::Bucket"

    # create a file in S3, which contains the secret from MySecretPassword
    MyS3File:
      Type: Custom::S3File
      Properties:
        # tell boto to create an s3 client
        Service: s3
        Create:
          # write the object when the resource is created
          Method: put_object
          KwArgs:
            # as body, use the attribute of the secret parameter
            Body: !GetAtt MySecretPassword.Value
            Bucket: !Ref MyS3Bucket
            Key: 'myPasswordFile.txt'
        # when deleting the resource, also cleanup the file from the bucket
        Delete:
          Method: delete_object
          KwArgs:
            Bucket: !Ref MyS3Bucket
            Key: 'myPasswordFile.txt'
        ServiceToken:
          "Fn::GetAtt":
            - BotoCustomResource
            - Arn

    # Get a secret parameter from the parameter store
    MySecretPassword:
      Type: Custom::SecretPassword
      Properties:
        Service: ssm
        Create:
          Method: get_parameter
          KwArgs:
            Name: '/my/secret/password'
            WithDecryption: 'True'
        ServiceToken:
          "Fn::GetAtt":
            - BotoCustomResource
            - Arn

    # Create the custom resource function which contains the calls to boto
    BotoCustomResource:
      Type: AWS::Lambda::Function
      Properties:
        Handler: "index.handler"
        Runtime: python3.6
        Timeout: 30
        Role: !GetAtt LambdaExecutionRole.Arn
        Code:
          ZipFile:
            "Fn::Join":
              - "\n"
              -
                - "import boto3"
                - "import json"
                - "import ast"
                - "from botocore.vendored import requests"
                - ""
                - "def flatten(d, parent_key=''):"
                - "    items = []"
                - "    for k, v in d.items():"
                - "        try:"
                - "            items.extend(flatten(v, '%s%s.' % (parent_key, k)).items())"
                - "        except AttributeError:"
                - "            items.append(('%s%s' % (parent_key, k), v))"
                - "    return dict(items)"
                - ""
                - "def handler(event, context):"
                - "    props = event['ResourceProperties']"
                - "    data = {}"
                - "    client=boto3.client(props['Service'])"
                - "    event_type = event['RequestType']"
                - "    status = 'SUCCESS'"
                - "    reason = ''"
                - "    data = {}"
                - ""
                - "    if event_type in ['Create', 'Delete', 'Update'] and event_type in props:"
                - "        method=getattr(client, props[event_type]['Method'])"
                - "        kwargs = {}"
                - "        for key, value in props[event_type]['KwArgs'].items():"
                - "            try:"
                - "                kwargs[key] = ast.literal_eval(value)"
                - "            except Exception as e:"
                - "                print(str(e))"
                - "                kwargs[key] = value"
                - "        try:"
                - "            data = method(**kwargs)"
                - "            status = 'SUCCESS'"
                - "            reason = ''"
                - "        except Exception as e:"
                - "            status = 'FAILED'"
                - "            reason = str(e)"
                - "            data = {}"
                - ""
                - "    response_data = {"
                - "        'Status': status,"
                - "        'Reason': reason,"
                - "        'PhysicalResourceId': event['LogicalResourceId'],"
                - "        'StackId': event['StackId'],"
                - "        'RequestId': event['RequestId'],"
                - "        'LogicalResourceId': event['LogicalResourceId'],"
                - "        'Data': flatten(data)"
                - "    }"
                - "    print(response_data)"
                - "    resp = requests.put(event['ResponseURL'], data=json.dumps(response_data))"
                - "    if resp.status_code != 200:"
                - "        print(resp.text)"
                - "        raise Exception(resp.text)"
                - "    return"

    # execution role for the custom resource function, needs S3 and SSM access
    LambdaExecutionRole:
      Type: AWS::IAM::Role
      Properties:
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Effect: Allow
            Principal:
              Service:
              - lambda.amazonaws.com
            Action:
            - sts:AssumeRole
        Path: "/"
        Policies:
        - PolicyName: root
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
            - Effect: Allow
              Action:
              - logs:CreateLogGroup
              - logs:CreateLogStream
              - logs:PutLogEvents
              Resource: arn:aws:logs:*:*:*
            - Effect: Allow
              Action:
                - s3:PutObject
                - s3:DeleteObject
                - ssm:GetParameter
              Resource: "*"
```

Using the Custom Resource attributes
-------------------------------
You can get the values(/attributes) of the response, eg: `!GetAtt MyS3File.ETag` will give you the ETag for the created file. Which you could then use as a value for whatever you want. For example in outputs:

```yml
Outputs:
  S3FileETag:
    Description: MyFileEta
    Value: !GetAtt MyS3File.ETag
```

Conclusion
----------
Using custom resources is always a hassle, but at least with the Custom Boto Resource, you can make things a lot easier to maintain, and make it more generic for re-usability. Also, because we can put the function inline the cloudformation script, we only have one place to maintain.
