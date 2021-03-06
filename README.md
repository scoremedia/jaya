# jaya
[Experimental][Seeking Feedback] Create data pipelines of AWS Services in Python.



## Installation
Currently, I expect a lot of iterations and hence hesitate to publish a pip versioned library. However, if you wish to play with it, you can

```bash
pip install git+ssh://git@github.com/scoremedia/jaya.git
```

# Example Code
Tested on 3.6+

```bash
# Let's make a client project
mkdir jaya-client
cd jaya-client
# Only tested on Python 3.6+
virtualenv -p python3 venv3
source venv3/bin/activate
# Install Jaya(See section `Installation`)
```

### Create the helper file
##### jaya-client/jayaclient/pipelines/copy_helper.py
```pythonstub
import boto3


def resource(conf, resource_name, region_name='us-east-1'):
    session = boto3.session.Session(aws_access_key_id=conf['aws_id'],
                                    aws_secret_access_key=conf['aws_key'],
                                    region_name=region_name)
    return session.resource(resource_name)


def copy_from_s3_to_s3(conf, source_bucket, source_key, destination_bucket, destination_key):
    s3 = resource(conf, 's3')
    o = s3.Object(destination_bucket, destination_key)
    o.copy_from(CopySource=source_bucket + '/' + source_key)


def get_bucket_key_pairs_from_event(event):
    return [(record['s3']['bucket']['name'],
             record['s3']['object']['key'])
            for record
            in event['Records']]


def copy_handler(aws_config, jaya_context, event, context):
    config
    print('Configuration Size:')
    print(len(aws_config))  # or print any value

    bucket_key_pairs = get_bucket_key_pairs_from_event(event)
    destination_buckets = [s3_child.bucket_name for s3_child in jaya_context.children()]

    for destination_bucket in destination_buckets:
        for source_bucket, source_key in bucket_key_pairs:
            copy_from_s3_to_s3(aws_config,
                               source_bucket,
                               source_key,
                               destination_bucket,
                               source_key)

```

### Copy Pipeline
##### jaya-client/jayaclient/pipelines/copy_pipeline.py
```pythonstub
from jaya import S3, Pipeline, AWSLambda

from jayaclient.pipelines import copy_helper
from jayaclient.config import config
# Note this import is for adding the AWSLambda dependencies
import jayaclient
from functools import partial

environment = 'development'
# I get my aws_id and aws_key in a `conf` dict
conf = config.get_aws_config(environment)
region = 'us-east-1'
pipeline_name = 'my-copy-pipeline'
lambda_name = 'CopyLambda'

s1 = S3(bucket_name='tsa-tmp-bucket1',
        region_name=region,
        events=[S3.event(S3.ALL_CREATED_OBJECTS, service_name=lambda_name)])

# copy_handler takes an additional config parameter which we can set right now before deployment
handler = partial(copy_helper.copy_handler, conf)

copy_lambda = AWSLambda(lambda_name,
                        handler,
                        region,
                        alias=environment,
                        virtual_environment_path='/Users/rabraham/Documents/dev/thescore/analytics/jaya-client/venv3/',
                        role_name='lambda_s3_exec_role',  # Existing role which has to be created manually
                        description="Hail Copy Handler",
                        dependencies=[jayaclient, copy_helper])

s2 = S3(bucket_name='tsa-tmp-bucket2', region_name=region)

p = s1 >> copy_lambda >> s2
piper = Pipeline(pipeline_name, [p])


```

The code piece `p = s1 >> copy_lambda >> s2` says 

* create `s1` and `s2` if it does not exist, create or update `copy_lambda`
* create an event notification such that if a file is created in `s1`, it will invoke `CopyLambda` and copy the file to `s2`.   

### Deploy the pipeline
```bash
jaya-client> PYTHONPATH=. jaya deploy --config_file=./jayaclient/config/jaya.conf --file=./jayaclient/pipelines/copy_pipeline.py --pipeline=my-copy-pipeline
```

The above code will create the S3 buckets if they don't exist.
If you go to your AWS Lambda Console, you'll see the deployed lambda. Check the alias `development` and you'll see the trigger for the S3 bucket. Likewise, if you go to the S3 Console for the bucket in `s1`, you'll see the event notification added for the lambda function and alias. 


### Redeploy a specific lambda function(e.g after making changes)
```bash
jaya-client> PYTHONPATH=. jaya deploy --config_file=./jayaclient/config/jaya.conf --file=./jayaclient/pipelines/s3_to_redshift_pipeline.py --pipeline=my-s3-to-redshift --function=CopyLambda
```

# Why Jaya?

Currently, we can specify our deployment as a `JSON` dictionary. For a very simple pipeline, check out the PSEUDO ABSOLUTELY INCORRECT CloudFormation JSON Dict
 
```json
{
    "AWSTemplateFormatVersion": "2010-09-09",

    "Resources": {
        "CopyRajiv": {
            "Type": "AWS::Lambda::Function",
            "Properties": {
                "Code": {
                    "S3Bucket": "thescore-tmp",
                    "S3Key": "CopyS3Lambda"
                },
                "FunctionName": "CopyS3Lambda",
                "Handler": "lambda.handler",
                "Runtime": "python3.6",
                "Timeout": 300,
                "Role": "arn:aws:iam::666:role/lambda_s3_exec_role",
            }
        },
        "SrcBucket": {
            "Type": "AWS::S3::Bucket",
            "Properties": {
                "BucketName": "tsa-rajiv-bucket1",
                "NotificationConfiguration": {
                    "LambdaConfigurations": [{
                        "Function": {"Ref": "CopyRajiv"},
                        "Event": "s3:ObjectCreated:*"
                    }]
                }
            }
        },
        "DestBucket": {
            "Type": "AWS::S3::Bucket",
            "Properties": {
                "BucketName": "tsa-rajiv-bucket2"
            }
        },
        "AliasForMyApp": {
            "Type": "AWS::Lambda::Alias",
            "Properties": {
                "FunctionName": "CopyRajiv",
                "FunctionVersion": "$LATEST",
                "Name": "staging"
            }
        },
        "LambdaInvokePermission": {
            "Type": "AWS::Lambda::Permission",
            "Properties": {
                "FunctionName": {"Fn::GetAtt": ["AliasForMyApp", "Arn"]},
                "Action": "lambda:InvokeFunction",
                "Principal": "s3.amazonaws.com",
                "SourceArn": {"Ref": "SrcBucket"}
            }
        }

    }

}

```

What if we could capture the same intent in Python: See the section `Elevator Pitch(Pseudocode)` for how the above would be expressed in `jaya`


The benefits of using `jaya`:
* We can see the flow of data through the pipeline more easily. We see that a `s1` bucket feeds into a `CopyLambda` which writes to a `s2` bucket. Granted that, we could compose the data too in the JSON dict. It may be personal opinion that the tree like syntax reads better. Imagine a complex multi-child tree.
```python
p = n1 >> n2 >> [n3 >> n4 >> [n7,
                              n8],
                 n5 >> n6]
```

* In the CloudFormation Script above, we just see that the lambda code was zipped and placed in an s3 bucket. How do we know which piece of code and from where. In the Python code above, we can use the `Goto Definition` feature in many editors and instantly look at the lambda code. We blur the line between functionality and deployment specific information. 

* We have a class which represents a lambda function i.e. `AWSLambda`. We now have a *language* to describe a Lambda as a Python class.

    - We can share AWSLambda in libraries. We could create a `S3ToFirehoseLambda` and share it!


## Supported Services

| From/To  | S3 | Lambda |
| ---- | ------------- |------|
| S3  | N/A  |Yes|
| Lambda  | Yes  |No|

## TODO
- Add Dead Letter Queue Support to `AWSLambda`
- Add Environment variables etc. to `AWSLambda`
- [Investigate] Automatically infer virtual environment path?
- Automatically create roles to let for e.g. the `AWSLambda` to read from an S3 bucket

