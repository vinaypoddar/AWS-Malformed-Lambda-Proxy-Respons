# AWS-Malformed-Lambda-Proxy-Response

## Background
I tried to call the Lambda function that creates an instance from the API gateway, but I made it next time, but I suffered from the following error.
The situation is that the test on Lambda succeeded and the test on the API gateway failed.

```log
# Response body
{
  "message": "Internal server error"
}

# Response header
{"x-amzn-ErrorType":"InternalServerErrorException"}

# log
Execution log for request ...
(summary)
Sat Sep 19 07:14:11 UTC 2020 : Received response. Status: 200, Integration latency: 2266 ms
Sat Sep 19 07:14:11 UTC 2020 : Endpoint response headers: ...
Sat Sep 19 07:14:11 UTC 2020 : Endpoint response body before transformations: ...
Sat Sep 19 07:14:11 UTC 2020 : Execution failed due to configuration error: Malformed Lambda proxy response
Sat Sep 19 07:14:11 UTC 2020 : Method completed with status: 502
```

At this point the lambda function is:

```python
import json
import boto3

def lambda_handler(event, context):
    region = event['pathParameters']['region']
    instances = [event['pathParameters']['instance']]
    ec2 = boto3.client('ec2', region_name = region)
    try:
        response = ec2.start_instances(InstanceIds=instances)
        return {
            'statusCode': 200,
            'body': response
        }
    except Exception as err:
        return {
            'statusCode': 500,
            'body': { 'message': f'{err}' }
        }
```

Give the region and instance ID as path parameters, and if you make a request with GET, the instance will start.

## What is Malformed Lambda proxy response

Please refer to the [official document](https://aws.amazon.com/jp/premiumsupport/knowledge-center/malformed-502-api-gateway) for detailed explanation .
Roughly speaking, Lambda's response format is not something that the API gateway can handle.

Rewrite the return of the response according to the above document.
Rewrite the response in case of an error in the same way.

```python
        return {
            'isBase64Encoded': False,
            'statusCode': 200,
            'headers': {},
            'body': response
        }
```

result...

```log
Execution failed due to configuration error: Malformed Lambda proxy response
Method completed with status: 502
```

Not yet. I get the same error. According to
[a similar case](https://stackoverflow.com/questions/47907641/malformed-lambda-proxy-response-from-aws-api-gateway-calling-a-lambda) of [StackOverflow](https://stackoverflow.com/questions/47907641/malformed-lambda-proxy-response-from-aws-api-gateway-calling-a-lambda), it seems that the body setting value must be a JSON string.
It was plainly written in the official document. .. ..

> If the body field returns JSON, it must be converted to a string to prevent further response issues. You can use JSON.stringify to handle this with Node.js functions. Other runtimes require different solutions, but the concept is the same.

That's why I convert the body to a JSON string.

```python
        return {
            'isBase64Encoded': False,
            'statusCode': 200,
            'headers': {},
            'body': json.dumps(response)
        }
```

results...


```log
Successfully completed execution
Method completed with status: 200
```

I passed! !!
Now you can launch an instance with just one access to the URL.

## Digression

Field that has been introduced as a format example of a response ( `isBase64Encoded`, `statusCode` etc.) is not any of the required parameters.
Therefore, `isBase64Encoded` it is not necessary to set when not exchanging binary data, and it is not necessary to set when there is no header to be added `headers`.
It seems that it is a field that can be set to the last.

## Final Lambda function

```python
import json
import boto3

def lambda_handler(event, context):
    region = event['pathParameters']['region']
    instances = [event['pathParameters']['instance']]
    ec2 = boto3.client('ec2', region_name = region)
    try:
        response = ec2.start_instances(InstanceIds=instances)
        return {
            'statusCode': 200,
            'body': json.dumps(response)
        }
    except Exception as err:
        return {
            'statusCode': 500,
            'body': json.dumps({
                'message': f'{err}'
            })
        }

```

Parameter errors should be handled with 4xx, but that's it for me.
