# iclab.simple
> A simple brick for building serverless applications

![IC Public Index version][badge-version]
![Development Status][badge-status]
[![License][badge-license]][license]
[![Slack][badge-slack]][ic-slack]

[ic.dev][ic-home] is an open source project that makes it easy to 
compose, share, and deploy cloud infrastructure bricks. This project
provides an [IC Public Index][ic-index] brick to build and secure 
serverless applications with AWS Lambda, Amazon API Gateway, Amazon 
Cognito, and various other resources such as Amazon DynamoDB, Amazon S3, 
etc.

## Preview

```python
# index.ic
from iclab import simple
from . import assets

@resource
def hello_world():
    api = simple.api("api", "demo", "0.1.0")
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])
    fun.http(api, "get", "/hello")
    return fun["routes"][0]
```
```javascript
// handler.js
exports.handle = async () => ({
    statusCode: 200,
    body: "Hello, World!\n"
})
```

```
$ ic aws up :hello_world preview
$ ic aws value preview | xargs curl
Hello, World!
```

## Usage
> If you haven't yet installed the [IC Command-Line Interface][ic-cli], 
check out the [Getting Started][ic-start] page.

<!-- --------------------------------------------------------------- -->
<details>
<summary>Create an AWS Lambda function</summary>

```python
from iclab import simple
from . import assets

@resource
def example():
    simple.function(
        "fun",
        # The identifier of the function's runtime.
        # - Must be one of nodejs10.x, nodejs8.10, python3.6, python3.7,
        #   python2.7, ruby2.5, java8, go1.x, dotnetcore2.1, dotnetcore1.0
        "nodejs8.10",
        # The code for the function.
        # - Must be an asset https://ic.dev/docs/en/assets
        # - The asset is automatically uploaded to your IC Amazon S3 bucket
        # - If the asset name ends with `.zip`, the default handler is
        #   handler.handle.
        # - If the asset is not a zip archive, the default handler is
        #   index.handle and the code of the handler is inlined.
        assets["handler.js"],
        # A dictionary of environment variables that are accessible from
        # function code during execution
        # - Defaults to None
        environ=dict(FOO="bar", BAZ="42"),
        # The function's execution role policies.
        # - By default, the AWS Lambda function is automatically assigned
        #   a custom AWS IAM role with the adequate trust policy and the
        #   AWSLambdaBasicExecutionRole managed policy.
        policies=[
            # To add an AWS managed policy, provide only its name
            "AmazonDynamoDBFullAccess",
            # To add a customer managed policy provide the full ARN
            "arn:aws:iam::xxxxxxxxxxxx:policy/MyPolicy",
            # To add an inline policy:
            dict(name="my_policy", effect="allow", action="*", resource="*"),
        ],
        # The number of days to retain the function's log events.
        # Defaults to 30
        log_retention=30,
        # You can also provide custom AWS Lambda function properties.
        # timeout: 60,
        # memory_size: 1024,
        # ...
    )
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Add an Amazon API Gateway endpoint</summary>

```python
from iclab import simple
from . import assets

@resource
def example():
    # Create an Amazon API Gateway
    api = simple.api(
        "api",
        # The name of the stage for the deployment.
        # - Example: test, prod, etc
        "demo",
        # The version of the API for the deployment.
        # - Example: 1.0.0 (without a leading v letter)
        # - Each time you add, remove, or modify a method or a path in 
        #   the API, you must also update this version.
        "0.1.0",
    )

    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Create an Amazon API Gateway endpoint for the function.
    fun.http(
        # The target Amazon API Gateway resource.
        api,
        # The HTTP method that clients use to call this endpoint.
        # - Can be any valid HTTP method like GET, POST, PUT, etc. or ANY.
        "GET",
        # The path that clients use to call this endpoint.
        # - Can be a nested path like /hello/world
        # - Can be a catch-all path like /hello/{proxy+}
        "/hello",
    )

    # Each time you add an endpoint to the function, the corresponding
    # route is appended to the function's routes attribute.
    # ["https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/demo/hello"]
    return fun["routes"] 
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Add a binary Amazon API Gateway endpoint</summary>

```javascript
// handler.js
const https = require("https");
const Buffer = require('buffer').Buffer

exports.handle = (event, context, callback) => {
  https.get("https://ic.dev/favicon.ico", res => {
    const chunks = [];
    res.on("data", chunk => chunks.push(Buffer.from(chunk)));
    res.on("end", () => {
      callback(null, {
        statusCode: 200,
        headers: { "Content-Type": "image/x-icon" },
        isBase64Encoded: true, // <==
        body: Buffer.concat(chunks).toString('base64'),
      })
    });
  });
}
```

```python
from iclab import simple
from . import assets

@resource
def example():
    # Create an Amazon API Gateway
    api = simple.api("api", "demo", "0.1.0")

    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Create an Amazon API Gateway endpoint for the function.
    fun.http(
      api, 
      "GET", 
      "/image", 
      # The list of binary MIME handled by this route.
      # - Use "*/*" if you need to serve images, etc inside a browser
      # - If you use specific 'image/jpeg' like MIME, be sure to query
      #   the API with an adequate 'Accept: image/jpeg' header.
      # - Defaults to None
      binary_media="*/*",
    )

    # Return the actual route to the function.
    return fun["routes"][0]
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Secure an Amazon API Gateway endpoint with Amazon Cognito</summary>

```python
from iclab import simple
from . import assets

@resource
def example(client_ids, user_pool_arn):
    # Create an Amazon API Gateway
    api = simple.api("api", "demo", "0.1.0")

    # Add an Amazon Cognito User Pool as an authorizer for the API.
    api.cognito(
        # A name to identify this particular authorizer in the API config.
        "main",
        # The list of Amazon Cognito app client ids valid for this 
        # authorizer.
        # - It can be a single client id: "abcd1234"
        # - It can be an array of client ids: ["abcd1234", "efgh56789"]
        client_ids,
        # The ARN of the Amazon Cognito User Pool
        user_pool_arn,
        # The name of the header containing the authorization token
        # - Defaults to 'Authorization'
        header="Authorization",
    )

    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Create an Amazon API Gateway endpoint for the function.
    fun.http(
        api,
        "GET",
        "/hello",
        # The name of the Amazon Cognito User Pool authorizer
        auth="main",
    )
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Add an Amazon DynamoDB table</summary>

```javascript
// handler.js
const AWS = require('aws-sdk');
const client = new AWS.DynamoDB.DocumentClient();

exports.handle = async () => {
  const data = await client.update({
    TableName: process.env.TABLE_EXAMPLE, // <==
    Key: {
      customer_id: 'c1',
      order_id: 'o1',
    },
    UpdateExpression: 'add #counter :value',
    ExpressionAttributeNames: {
      '#counter': 'counter',
    },
    ExpressionAttributeValues: {
      ':value': 1,
    },
    ReturnValues: 'ALL_NEW',
  }).promise()

  return {
    statusCode: 200,
    body: `${data["Attributes"]["counter"]}\n`
  }
}
```

```python
from iclab import simple
from . import assets

@resource
def example():
    # Create an Amazon API Gateway
    api = simple.api("api", "demo", "0.1.0")

    # Create an Amazon DynamoDB table
    table = simple.table(
        "table",
        # Specify 'name' as a partition key of type 'string'
        # - Valid types are binary, boolean, binary_set, list, map, 
        #   number, number_set, null, string, or string_set.
        ("customer_id", "string"),
        # If you want a sort key, otherwise omit.
        # - Defaults to None
        ("order_id", "string"),
        # If you want the table to stream its changes, otherwise omit.
        # - Valid values are NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES,
        #   KEYS_ONLY
        # - Defaults to None
        stream="NEW_IMAGE",
        # If you want to use a provisioned scaling model managed by
        # Amazon CloudWatch, otherwise omit.
        # - Defaults to a pay per request scaling model managed
        #   automatically by Amazon DynamoDB.
        scaling=dict(
            # The read and write capacities default to [5-40000]. They
            # follow a target tracking scaling policy based on 70% of
            # the Amazon DynamoDB capacity utilization.
            read=dict(min=5, max=40000, target=70),
            write=dict(min=5, max=40000, target=70),
        ),
    )

    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Connect the table to the function.
    fun.table(
        # The target Amazon DynamoDB table.
        table,
        # A name to identify this particular table in the function's
        # config.
        # - An environment variable named TABLE_EXAMPLE is automatically
        #   made available for the function. Its value is the name of
        #   the table.
        "example",
        # To control the permissions granted to the function, provide
        # only the name of the desired actions.
        # - Default to '*'
        actions=["PutItem", "UpdateItem"],
    )

    # Create an Amazon API Gateway endpoint for the function.
    fun.http(api, "GET", "/counter")

    # Return the actual route to the function.
    return fun["routes"][0]
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Listen to an Amazon DynamoDB stream</summary>

```javascript
// handler.js
exports.handle = async (event, context) => {
  event.Records.forEach(function (record) {
    console.log(record.dynamodb);
  })
}
```

```python
from iclab import simple
from . import assets

@resource
def example():
    # Create an Amazon DynamoDB table
    table = simple.table("table", ("id", "string"), stream="NEW_IMAGE")

    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Connect the table to the function.
    fun.table_stream(
        # The target Amazon DynamoDB table.
        table,
        # The position in a stream from which to start reading.
        # - Valid values are TRIM_HORIZON and LATEST.
        # - Defaults to LATEST
        position="LATEST",
        # The maximum number of items to retrieve in a single batch.
        # - Default to 100
        # - Maximum 1000
        batch=100,
        # Disables the event source mapping to pause polling and
        # invocation.
        # - Default to True
        enabled=True,
    )

    # Return the name of the function
    return fun["function"]["ref"]
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Add an Amazon S3 bucket</summary>

```javascript
// handler.js
const AWS = require('aws-sdk');
const client = new AWS.S3();

exports.handle = async (event, context) => {
  await client.putObject({
    Body: new Buffer(event.body, 'base64'),
    Bucket: process.env.STORAGE_EXAMPLE, // <==
    Key: "example",
  }).promise();

  return { statusCode: 200 }
}
```

```python
from ic import aws
from iclab import simple
from . import assets


@resource
def example():
    # Create an Amazon API Gateway
    api = simple.api("api", "demo", "0.2.0")

    # Create an Amazon S3 bucket
    bucket = aws.s3.bucket("bucket")

    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Connect the bucket to the function.
    fun.storage(
        # The target Amazon S3 bucket.
        bucket,
        # A name to identify this particular bucket in the function's
        # config.
        # - An environment variable named STORAGE_EXAMPLE is
        #   automatically made available for the function. Its value is
        #   the name of the bucket.
        "example",
        # To control the permissions granted to the function, provide
        # only the name of the desired actions.
        # - Default to '*'
        actions=["PutObject", "GetObject"],
    )

    # Create an Amazon API Gateway endpoint for the function.
    fun.http(api, "POST", "/upload", binary_media="*/*")

    # Return the actual route to the function.
    return fun["routes"][0]
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Listen to an Amazon S3 bucket</summary>

```javascript
// handler.js
exports.handle = async (event, context) => {
  event.Records.forEach(function (record) {
    console.log(record.s3);
  })
}
```

```python
from ic import aws
from iclab import simple
from . import assets


@resource
def example():
    # Create an Amazon S3 bucket
    bucket = aws.s3.bucket("bucket")

    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Connect the bucket to the function.
    fun.storage_events(
        # The target Amazon S3 bucket.
        bucket,
        # The list of Amazon S3 events to listen.
        # - Do not specify the 's3:' prefix
        # - See also 'Supported Event Types' in
        #   https://docs.aws.amazon.com/AmazonS3/latest/dev/NotificationHowTo.html
        events=["ObjectCreated:*"],
    )

    # Return the name of the function
    return fun["function"]["ref"]
```

</details>
<!-- --------------------------------------------------------------- -->
<details>
<summary>Add an AWS Secrets Manager secret</summary>

```javascript
// handler.js
const AWS = require('aws-sdk');
const client = new AWS.SecretsManager();

exports.handle = async () => {
  const secret = await client.getSecretValue({
    SecretId: process.env.SECRET_EXAMPLE, // <==
  }).promise();
  console.log(secret)
}
```

```python
from ic import aws
from iclab import simple
from . import assets


@resource
def example():
    # Create an AWS Lambda function
    fun = simple.function("fun", "nodejs8.10", assets["handler.js"])

    # Create a AWS Secrets Manager secret
    secret = aws.secrets_manager.secret("secret", name="/my/secret")

    # Connect the secret to the function.
    fun.secret(
        # The target AWS Secrets Manager secret.
        secret,
        # A name to identify this particular secret in the function's
        # config.
        # - An environment variable named SECRET_EXAMPLE is automatically
        #   made available for the function. Its value is the id of
        #   the secret.
        "example",
        # To control the permissions granted to the function, provide
        # only the name of the desired actions.
        # - Default to '*'
        actions=["GetSecretValue"],
    )

    # Return the name of the function
    return fun["function"]["ref"]
```

</details>
<!-- --------------------------------------------------------------- -->

## License

Copyright 2019 Farzad Senart and Lionel Suss. All rights reserved.

Unless otherwise stated, this project is licensed under the [Apache 
License Version 2.0][license-file].

[badge-version]: https://img.shields.io/badge/ic-v0.3.0-orange
[badge-status]: https://img.shields.io/badge/status-beta-yellow
[badge-license]: https://img.shields.io/badge/license-Apache--2.0-green
[badge-slack]: https://slack.ic.dev/badge.svg
[ic-home]: https://ic.dev
[ic-index]: https://ic.dev/docs/en/community
[ic-cli]: https://github.com/icdotdev/cli
[ic-start]: https://ic.dev/docs/en/installation
[ic-slack]: https://slack.ic.dev
[license]: https://github.com/icdotdev/iclab.simple#license
[license-file]: https://github.com/icdotdev/iclab.simple/blob/master/LICENSE