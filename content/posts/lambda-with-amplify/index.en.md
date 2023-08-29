---
title: "Call AWS Lambda function with AWS Amplify directly"
date: 2023-08-27
draft: false
tags: ["AWS", "Lambda", "Amplify"]
---

In this comprehensive guide, we will explore how to execute AWS Lambda from your code using AWS Amplify.

The power of AWS Amplify in helping developers create feature-rich, full-stack web and mobile applications in a matter 
of hours. Whether you're a frontend developer or a mobile app enthusiast, AWS Amplify provides an easy-to-use solution 
that allows you to build, ship, and host applications on AWS, leveraging the wide range of AWS services as your use 
cases evolve. With no need for cloud expertise, Amplify empowers you to create a robust backend, visually build a 
frontend UI, and deploy your app seamlessly.

As for date when the article is written, Amplify provides an interface to some categories of AWS services such as 
Storage, API, GEO, Analytics, etc.

As a developer, sometimes you need to abstract out some code due to you want to use library or SDK which is not available
for the platform or programming language you use. And the easiest way of doing that is to put this code in AWS Lambda
and call this lambda from your code. Or you may have some lambda code you want to call from your mobile app or UI code.

Unfortunately, AWS Amplify does not support calling AWS Lambda directly. You can put a lambda function behind AWS API 
Gateway and call API Gateway from Amplify.  

What if this would be a way to call AWS Lambda function directly without setting up an additional layer of code and
call execution?

And there is such a way. It might not be straight-forward, but definitely possible. 

## Lambda's Invoke endpoint

You can invoke Lambda functions directly using function URL HTTP(S) endpoint.
A function URL is an essential component of your Lambda function that serves as a dedicated endpoint for HTTP(S) requests. 
This endpoint allows your Lambda function to be invoked via HTTP(S) requests from various sources.
You can create and configure a function URL through the Lambda console or the Lambda API. When you create a function URL, 
Lambda automatically generates a unique URL endpoint for you. This endpoint is exclusive to your function URL, and no 
other function shares the same endpoint.
Once you create a function URL, its URL endpoint remains constant. You can rely on this endpoint to call your Lambda 
function consistently. 

Function URL endpoints have the following format:
```
https://<url-id>.lambda-url.<region>.on.aws
```

But also you can skip creation of a function URL HTTP(S) endpoint and use Lambda's Invoke endpoint directly:

```
https://lambda.${region}.amazonaws.com/2015-03-31/functions/${functionName}/invocations
```

This would require that you sign the request using the AWS account credentials.

## Using AWS Amplify to sign any service HTTPS requests 

We can use the Amplify API category, to make a REST request to Lambda's Invoke endpoint. But we also need to sign our
request. We will use Amplify Auth category to do it. Let's prepare Amplify configuration:

{{< labelled-highlight lang="json" filename="amplifyconfiguration.json" >}}
{
    "UserAgent": "aws-amplify-cli/2.0",
    "Version": "1.0",
    "auth": {
        "plugins": {
            "awsCognitoAuthPlugin": {
                "UserAgent": "aws-amplify-cli/0.1.0",
                "Version": "0.1.0",
                "IdentityManager": {
                    "Default": {}
                },
                "CredentialsProvider": {
                    "CognitoIdentity": {
                        "Default": {
                            "PoolId": "${region}:${identity-pool-id}",
                            "Region": "${region}"
                        }
                    }
                },
                "CognitoUserPool": {
                    "Default": {
                        "PoolId": "${user-pool-id}",
                        "AppClientId": "${app-client-id}",
                        "Region": "${region}",
                        "AppClientSecret": "${app-client-secret}"
                    }
                },
                "Auth": {
                    "Default": {
                        "authenticationFlowType": "USER_SRP_AUTH"
                    }
                }
            }
        }
    },
    "api": {
        "plugins": {
            "awsAPIPlugin": {
                "lambdaExecution": {
                    "name": "lambdaExecution",
                    "endpointType": "REST",
                    "endpoint": "https://lambda.${region}.amazonaws.com/2015-03-31/functions/${functionName}/invocations",
                    "region": "${region}",
                    "authorizationType": "NONE"
                }
            }
        }
    }
}
{{</ labelled-highlight >}}

Configuration consists of 2 parts:
- Cognito Auth plugin
- API plugin

### Amplify Cognito Auth Plugin
We use Cognito auth plugin to get temporary AWS credentials using authenticated user or guest credentials, which later 
we can use to sign our request.

Amazon Cognito identity pools provide temporary AWS credentials for user who are guests (unauthenticated) and for users 
who have been authenticated and received a token. An identity pool is a store of user identity data specific to your 
account.

Create identity pool which later you can reference in Amplify configuration.
Amazon Cognito identity pools support both authenticated and unauthenticated identities. Authenticated identities belong 
to users who are authenticated by any supported identity provider. Unauthenticated identities typically belong to guest 
users. Set up access of your choice. 

Please note that authenticated identities are preferred as a more secure option.

You can use Cognito user pool as an identity provider and create user in this user pool, later you use to authenticate,
and add appropriate configuration settings for Amplify as used in an example above.

Either you choose to use authenticated or guest access, you need to associate a proper IAM role in your identity pool.
An IAM role defines the permissions for your users to access AWS resources, like AWS Lambda. Users of your application 
will assume the roles you create. You can specify different roles for authenticated and unauthenticated users.

As our goal is to be able to execute AWS Lambda from our application, set up IAM role should have 
```"lambda:InvokeFunction"``` policy.

We have everything set up to sign the request. Now let's use Amplify API plugin and sign our request.

### Amplify API plugin
The Amplify API plugin provides an interface for making requests to your backend. Using REST version we can make a
REST requests to HTTP(S) endpoint. And this is exactly what we need. To execute AWS Lambda function, we need to make a
REST request to Lambda's end point. But as I wrote above this request needs to be signed and caller should use appropriate 
IAM role which allows to execute a Lambda function. 

API plugin supports multiple authorization types, such as ```AWS_IAM```, ```API_KEY```, ```AMAZON_COGNITO_USER_POOLS```
and ```NONE```.

And this sounds exactly what we need, and we can use either ```AWS_IAM``` or ```AMAZON_COGNITO_USER_POOLS``` authorization
to access a Lambda function. But as you might notice, I put ```NONE``` as an authorization type in Amplify settings for 
API plugin and this is for a reason. 

By default, Amplify API plugin is designed to make requests to AWS API Gateway and
by default all requests are signed with AWS API Gateway service name ```execute-api``` and this will not work for AWS
Lambda. We need to change it and use a custom signer for our API settings.

{{< labelled-highlight lang="kotlin" filename="CustomPlugin.kt" >}}
val customApiPlugin = AWSApiPlugin.builder()
    .configureClient("lambdaExecution") { okHttpBuilder ->
        okHttpBuilder.addInterceptor { chain ->
            val apiAuthProviders = ApiAuthProviders.noProviderOverrides()
            val credentialsProvider: CredentialsProvider =
            apiAuthProviders.awsCredentialsProvider ?: CognitoCredentialsProvider()
            val signer = ApiGatewayIamSigner("${region}")
            val serviceName = "lambda"
            val decorator =
            IamRequestDecorator(signer, credentialsProvider, serviceName)
            chain.proceed(decorator.decorate(chain.request()))
        }
    }
    .build()
{{</ labelled-highlight >}}

And finally we can add both plugins and call our AWS Lambda function.

{{< labelled-highlight lang="kotlin" filename="Example.kt" >}}
try {
    Amplify.addPlugin(AWSCognitoAuthPlugin())
    val customApiPlugin = AWSApiPlugin.builder()
        .configureClient("lambdaExecution") { okHttpBuilder ->
        okHttpBuilder.addInterceptor { chain ->
            val apiAuthProviders = ApiAuthProviders.noProviderOverrides()
            val credentialsProvider: CredentialsProvider =
            apiAuthProviders.awsCredentialsProvider ?: CognitoCredentialsProvider()
            val signer = ApiGatewayIamSigner("${region}")
            val serviceName = "lambda"
            val decorator =
            IamRequestDecorator(signer, credentialsProvider, serviceName)
            chain.proceed(decorator.decorate(chain.request()))
        }
    }
    .build()
    Amplify.addPlugin(customApiPlugin)
    Amplify.configure(applicationContext)
    Log.i("MyAmplifyApp", "Initialized Amplify")
} catch (e: AmplifyException) {
    Log.e("MyAmplifyApp", "Could not initialize Amplify", e)
}

Amplify.API.post("lambdaExecution", options,
    {
        Log.i("MyAmplifyApp", "POST succeeded: ${it.data.asString()}")
    },
    {
        Log.e("MyAmplifyApp", "POST failed", it)
    }
)
{{</ labelled-highlight >}}

In conclusion, with minor modifications we made AWS Amplify working with AWS Lambda functions directly without setting
up AWS API Gateway in front.

Happy coding!
