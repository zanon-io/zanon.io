Title: Serverless Notifications on AWS
Summary: A tutorial to implement the Pub/Sub pattern using AWS IoT.
Tags: Serverless, AWS, Node
Date: NOV 05, 2016
URL: serverless-notifications-on-aws

## TL;DR

This post is a tutorial that uses the Serverless Framework and AWS IoT for Serverless Notifications.

Demo: [serverless-notifications.zanon.io](https://serverless-notifications.zanon.io)

Code: [GitHub](https://github.com/zanon-io/serverless-notifications)

## Serverless Notifications

Real-time notifications are an important use case for modern apps. For example, you may need to notify your user that there is another post available in his social feed or that someone else added a comment in one of his photos.

Implementing notifications is an easy task when you use WebSockets and have a dedicated server. You can make a permanent link between the user and the server and use the publish-subscribe pattern to share messages. The browser will subscribe to automatically receive new messages without needing a polling mechanism to constantly check for updates.

But if we are going serverless, we don't have a dedicated server. Instead, we need a cloud service that will solve this problem for us providing scalability, high availability and charging per messages and not per hour.

In this post, I'm going to describe how I've implemented a notification system for unauthenticated users using the Serverless Framework and the AWS IoT for *browsers*. I know that "Internet of Things" sounds strange to be used in a website, but it supports WebSockets and is very easy to use. Besides, Amazon SNS (Simple Notification Service) has a better name, but doesn't support WebSockets.

IoT is used due to its simple messaging system. You create a "topic" and make users to subscribe to it. A message sent to this topic will be automatically shared with all subscribed users. A common use case for this is a chat system.

If you want private messages, you just need to create private topics and restrict the access. Only one user will be subscribed to this topic and you can make your system (Lambda functions) to send updates to this topic to notify this specific user.

### Architecture

In this tutorial, we are going to implement the following architecture.

![architecture](https://zanon.io/images/posts/2016-11-05-architecture.png)

1. User makes a request to Route 53 that is configured to reference a S3 bucket.

2. S3 bucket provides the frontend code (HTML / CSS / JavaScript / images) and the IoT client code.

3. After loading the frontend code, an Ajax request is done to the API Gateway to retrieve temporary keys.

4. The API Gateway redirects the request to be handled by a Lambda function.

5. The Lambda function connects to IAM to assume a role and create temporary AWS keys.

6. Frontend code subscribe to IoT events using the temporary keys.

> **Note**: instead of using API Gateway and Lambda to retrieve IAM credentials, you can use Cognito. More on that in the end of this post. 

### Frontend

Configuring Route 53 and Amazon S3 to serve static files is a common use case. I've already covered how to do this in another blog post. You can take a look [here](https://zanon.io/posts/building-serverless-websites-on-aws-tutorial#host-your-website) if you want.

In our frontend code, let's start creating the **index.html** for our layout.

``` html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Serverless Notifications</title>
        <link href="bootstrap.min.css" rel="stylesheet">
    </head>
    <body>
        <div class="container">
            <div class="row">
                <h2>Serverless Notifications</h2>
            </div>
            <div class="row">
                <input id="btn-keys" type="button" class="btn btn-primary" value='Retrieve Keys'>
                <input id="btn-connect" type="button" class="btn btn-primary" value='Connect'>
            </div>
            <div class="row">
                <input id="message" type="text">
                <input id="btn-send" type="button" class="btn btn-primary" value='Send Message'>
            </div>
            <div class="row">
                <h3>Log Messages</h3>
                <ul id="log"></ul>
            </div>
        </div>    
        <script src="jquery.min.js"></script>
        <script type="text/javascript">
            window.lambdaEndpoint = 'https://abcdefghij.execute-api.us-east-1.amazonaws.com/dev/iot/keys';
        </script>
        <script src="bundle.js"></script> <!-- IoT -->
    </body>
</html>
```

**Note**: I've created a `window.lambdaEndpoint` variable. You need to change this value later with the output of the Serverless Framework. This address will be your function endpoint.

This HTML results in:

[![layout](https://zanon.io/images/posts/2016-11-05-layout.png)](https://serverless-notifications.zanon.io)

In this demo, the user will click on "Retrieve Keys" to make an Ajax call to Lambda with the objective to retrieve temporary keys to connect with our IoT messaging system. With those keys, "Connect" will create a channel and subscribe to messages. The "Send Message" button will share messages with other users (open another browser tab to test).

### Request keys to Lambda

We need to make an Ajax call to a Lambda function to retrieve temporary AWS keys.

**Note**: I'm using ES6 syntax because this file will be processed by Babel later when I bundle with the IoT client.

``` javascript
$(document).ready(() => {

    let iotKeys;

    $('#btn-keys').on('click', () => {
        $.ajax({
            url: window.lambdaEndpoint,
            success: (res) => {
                addLog(`Endpoint: ${res.iotEndpoint},
                        Region: ${res.region},
                        AccessKey: ${res.accessKey},
                        SecretKey: ${res.secretKey},
                        SessionToken: ${res.sessionToken}`);

                iotKeys = res; // save the keys
            }
        });
    });     
});

const addLog = (msg) => {
    const date = (new Date()).toTimeString().slice(0, 8);
    $("#log").prepend(`<li>[${date}] ${msg}</li>`);
}
```

### AWS IoT

To build our notification system, we need to use a Node module [AWS IoT SDK](https://github.com/aws/aws-iot-device-sdk-js) and make a bundle to use in the browser.

In this project, I've created another folder named as **iot** to develop the IoT client. It has a **package.json**, so run `npm install` to install the **aws-iot-device-sdk** dependency.

The IoT object has the functions `connect` and `send`. It also offers handlers to other functions like `onConnect` and `onMessage`.

``` javascript
const awsIot = require('aws-iot-device-sdk');

let client, iotTopic;
const IoT = {

    connect: (topic, iotEndpoint, region, accessKey, secretKey, sessionToken) => {

        iotTopic = topic;

        client = awsIot.device({
            region: region,
            protocol: 'wss',
            accessKeyId: accessKey,
            secretKey: secretKey,
            sessionToken: sessionToken,
            port: 443,
            host: iotEndpoint
        });

        client.on('connect', onConnect);
        client.on('message', onMessage);            
        client.on('close', onClose);     
    },

    send: (message) => {
        client.publish(iotTopic, message);
    }  
};

const onConnect = () => {
    client.subscribe(iotTopic);
    addLog('Connected');
};

const onMessage = (topic, message) => {
    addLog(message);
};

const onClose = () => {
    addLog('Connection failed');
};
```

As we have developed the IoT client, we can now finish the JavaScript browser code. Add the following:

``` javascript
$('#btn-connect').on('click', () => {
    const iotTopic = '/serverless/pubsub';        

    IoT.connect(iotTopic,
                iotKeys.iotEndpoint,
                iotKeys.region,
                iotKeys.accessKey,
                iotKeys.secretKey,
                iotKeys.sessionToken);
});    

$('#btn-send').on('click', () => {
    const msg = $('#message').val();
    IoT.send(msg);    
});    
```

Now, let's create a bundle. It seems that currently this module can't be bundled with webpack, but with Browserify it worked fine. I've created a file named **make-bundle.js** that helps with this. You can take a look in this file [here](https://github.com/zanon-io/serverless-notifications/blob/master/iot/make-bundle.js).

Just run `npm install` inside this folder and run `node make-bundle` to create an updated version. This **bundle.js** will be used in our frontend code and must be in the same folder of the **index.html** file.

### Create an IoT Role

Our Lambda function will be responsible by creating temporary AWS keys. However, it needs a role to define what access those keys will provide.

You can create this role using the IAM console or execute the **index.js** file that is inside the **create-role** folder to create one for you. This package uses the AWS SDK and requires a `npm install` before using.

I've used the role name **serverless-notifications**. If you want, you can change this name here, but you will also need to change the name inside the Lambda function.

Create Role:

``` javascript
const AWS = require('aws-sdk');
const iam = new AWS.IAM();
const sts = new AWS.STS();
const roleName = 'serverless-notifications';

// get the account id
sts.getCallerIdentity({}, (err, data) => {
  if (err) return console.log(err, err.stack);

  const createRoleParams = {
    AssumeRolePolicyDocument: `{
      "Version":"2012-10-17",
      "Statement":[{
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::${data.Account}:root"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }`,
    RoleName: roleName
  };

  // create role
  iam.createRole(createRoleParams, (err, data) => {
    if (err) return console.log(err, err.stack);

    const attachPolicyParams = {
      PolicyDocument: `{
        "Version": "2012-10-17",
        "Statement": [{
          "Action": ["iot:Connect", "iot:Subscribe", "iot:Publish", "iot:Receive"],
          "Resource": "*",
          "Effect": "Allow"
        }]
      }`,
      PolicyName: roleName,
      RoleName: roleName
    };

    // add iot policy
    iam.putRolePolicy(attachPolicyParams, (err, data) => {
      if (err) console.log(err, err.stack);
      else     console.log(`Finished creating IoT Role: ${roleName}`);          
    });
  });
});
```

### Serverless Framework

To finish, let's create a Lambda function that will generate temporary keys (valid for 1 hour) to connect to the IoT service. We are going to use the Serverless Framework to help here. If you don't know how to use it, you can take a look [here](https://zanon.io/posts/building-serverless-websites-on-aws-tutorial) for another tutorial that I've created.

The **serverless.yml** must add Lambda permissions for `iot:DescribeEndpoint` (find your account endpoint) and `sts:AssumeRole` (create temporary keys).

``` xml
service: serverless-notifications

provider:
  name: aws
  runtime: nodejs4.3
  stage: dev
  region: us-east-1
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - 'iot:DescribeEndpoint'
      Resource: "*"
    - Effect: "Allow"
      Action:
        - 'sts:AssumeRole'
      Resource: "*"

functions:
  auth:
    handler: handler.auth
    events:
      - http: GET iot/keys
    memorySize: 128
    timeout: 10

package:
  exclude:
    - .git/**
    - create-role/**
    - frontend/**
    - iot/**
```

Lambda function:

``` javascript
'use strict';

const AWS = require('aws-sdk');
const iot = new AWS.Iot();
const sts = new AWS.STS();
const roleName = 'serverless-notifications';

module.exports.auth = (event, context, callback) => {

    // get the endpoint address
    iot.describeEndpoint({}, (err, data) => {
        if (err) return callback(err);

        const iotEndpoint = data.endpointAddress;
        const region = getRegion(iotEndpoint);

        // get the account id which will be used to assume a role
        sts.getCallerIdentity({}, (err, data) => {
            if (err) return callback(err);

            const params = {
                RoleArn: `arn:aws:iam::${data.Account}:role/${roleName}`,
                RoleSessionName: getRandomInt().toString()
            };

            // assume role returns temporary keys
            sts.assumeRole(params, (err, data) => {
                if (err) return callback(err);

                const res = buildResponseObject(iotEndpoint,
                                                region,
                                                data.Credentials.AccessKeyId,
                                                data.Credentials.SecretAccessKey,
                                                data.Credentials.SessionToken);
                callback(null, res);
            });
        });
    });
};

const buildResponseObject = (iotEndpoint, region, accessKey, secretKey, sessionToken) => {
    return {
        statusCode: 200,
        headers: {
            'Access-Control-Allow-Origin': '*'
        },  
        body: JSON.stringify({
            iotEndpoint: iotEndpoint,
            region: region,
            accessKey: accessKey,
            secretKey: secretKey,
            sessionToken: sessionToken
        })
    };
};

const getRegion = (iotEndpoint) => {
    const partial = iotEndpoint.replace('.amazonaws.com', '');
    const iotIndex = iotEndpoint.indexOf('iot');
    return partial.substring(iotIndex + 4);
};

// Get random Int
const getRandomInt = () => {
    return Math.floor(Math.random() * Number.MAX_SAFE_INTEGER);
};
```
### Testing

[![test](https://zanon.io/images/posts/2016-11-05-test.png)](https://serverless-notifications.zanon.io)

### Pricing

How much does it cost? Only $5 per million messages (USA). It can be pretty cheap depending on your scale and traffic because you don't need to pay for a dedicated server.

Official pricing page for IoT: https://aws.amazon.com/iot/pricing/

### Improving this demo

This sample considers that there is only one topic. It's ok if everyone is subscribed to the same channel. However, if you want to send private notifications to a specific user, you need to create a new topic per user.

To handle permissions, I suggest that you keep creating just one role with access to all topics (as we have done in the **create-role** project) and use the `assumeRole` command (inside Lambda) to create keys with restricted access with a specific topic. This restriction is done passing a policy document as one of the assumeRole parameters (see [docs](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/STS.html#assumeRole-property)).

For authenticated users, I suggest that you use Cognito credentials.

### What more?

I've tried another experiment with this and created a serverless multiplayer game sample. If you want to develop an HTML5 game in a serverless architecture, you can use IoT to change messages between players and implement a cheap multiplayer game. The performance is good enough for dynamic games.

Multiplayer Game: [demo](https://bombermon.zanon.io) and [code](https://github.com/zanon-io/serverless-multiplayer-game)  
(you can use your desktop and phone to test the multiplayer feature)

### Conclusion

This demo solved the use case of real-time notifications for serverless applications. If you have any doubts, feel free to post in comments.

### UPDATE: Nov 21, 2016

> [Ben Kehoe](https://twitter.com/ben11kehoe) suggests that Cognito can be easier than using IAM (with APIG + Lambda, like in this example). I've tried and found it to be similar in complexity when dealing with unauthenticated access. However, Cognito is **much** more powerful for *authenticated* access. If you need to support authenticated users, use Cognito.

Below follows how to use Cognito for *unauthenticated* access.

1) Create an Identity Pool

``` javascript
const AWS = require('aws-sdk');
const cognitoidentity = new AWS.CognitoIdentity();

const params = {
  AllowUnauthenticatedIdentities: true,
  IdentityPoolName: 'serverless-notifications'  
};

cognitoidentity.createIdentityPool(params, (err, data) => {
  if (err) console.log(err, err.stack);
  else     console.log(data.IdentityPoolId); // save the IdentityPoolId
});
```

2) Create a Role for this Identity Pool like we've done before. However, for the `AssumeRolePolicyDocument`, use the following JSON (set the IdentityPoolId with the value returned in the previous function).

``` javascript
`{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "cognito-identity.amazonaws.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "cognito-identity.amazonaws.com:aud": ${IdentityPoolId}
        },
        "ForAnyValue:StringLike": {
          "cognito-identity.amazonaws.com:amr": "unauthenticated"
        }
      }
    }
  ]
}`
```

3) Set the Identity Pool with the new Role

``` javascript
const AWS = require('aws-sdk');
const cognitoidentity = new AWS.CognitoIdentity();

const params = {
  IdentityPoolId: 'IDENTITY_POOL_ID',
  Roles: { 
    unauthenticated: 'ROLE_ARN',
	authenticated: 'ROLE_ARN'
  }
};

cognitoidentity.setIdentityPoolRoles(params, (err, data) => {
  if (err) console.log(err, err.stack);
  else     console.log(data); // successful response returns an empty object
});
```

4) In the browser, request the Cognito credentials

``` javascript
const AWS = require('aws-sdk');

AWS.config.region = 'YOUR_COGNITO_REGION';
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
    IdentityPoolId: 'IDENTITY_POOL_ID',
});

AWS.config.credentials.get(() => {

    // Use these credentials with IoT
    const accessKeyId = AWS.config.credentials.accessKeyId;
    const secretAccessKey = AWS.config.credentials.secretAccessKey;
    const sessionToken = AWS.config.credentials.sessionToken;
});
```