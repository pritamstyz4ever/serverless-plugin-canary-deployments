[![npm version](https://badge.fury.io/js/serverless-plugin-canary-deployments.svg)](https://badge.fury.io/js/serverless-plugin-canary-deployments)

# Serverless Plugin Canary Deployments

A Serverless plugin to implement canary deployments of Lambda functions, making use of the [traffic shifting feature](https://docs.aws.amazon.com/lambda/latest/dg/lambda-traffic-shifting-using-aliases.html) in combination with [AWS CodeDeploy](https://docs.aws.amazon.com/lambda/latest/dg/automating-updates-to-serverless-apps.html)

## Installation

`npm i --save-dev serverless-plugin-canary-deployments`

## Usage

To enable gradual deployments for Lambda functions, your `serverless.yml` should look like this:

```yaml
service: canary-deployments
provider:
  name: aws
  runtime: nodejs6.10
  iamRoleStatements:
    - Effect: Allow
      Action:
        - codedeploy:*
      Resource:
        - "*"

plugins:
  - serverless-plugin-canary-deployments

functions:
  hello:
    handler: handler.hello
    events:
      - http: GET hello
    deploymentSettings:
      type: Linear10PercentEvery1Minute
      alias: Live
      preTrafficHook: preHook
      postTrafficHook: postHook
      alarms:
        - FooAlarm
        - BarAlarm
  preHook:
    handler: hooks.pre
  postHook:
    handler: hooks.post
```

You can see a working example in the [example folder](./example/).

## Configuration

* `type`: (required) defines how the traffic will be shifted between Lambda function versions. It must be one of the following:
  - `Canary10Percent5Minutes`: shifts 10 percent of traffic in the first increment. The remaining 90 percent is deployed five minutes later.
  - `Canary10Percent10Minutes`: shifts 10 percent of traffic in the first increment. The remaining 90 percent is deployed 10 minutes later.
  - `Canary10Percent15Minutes`: shifts 10 percent of traffic in the first increment. The remaining 90 percent is deployed 15 minutes later.
  - `Canary10Percent30Minutes`: shifts 10 percent of traffic in the first increment. The remaining 90 percent is deployed 30 minutes later.
  - `Linear10PercentEvery1Minute`: shifts 10 percent of traffic every minute until all traffic is shifted.
  - `Linear10PercentEvery2Minutes`: shifts 10 percent of traffic every two minutes until all traffic is shifted.
  - `Linear10PercentEvery3Minutes`: shifts 10 percent of traffic every three minutes until all traffic is shifted.
  - `Linear10PercentEvery10Minutes`: shifts 10 percent of traffic every 30 minutes until all traffic is shifted.
* `alias`: (required) name that will be used to create the Lambda function alias.
* `preTrafficHook`: (optional) validation Lambda function that runs before traffic shifting. It must use te CodeDeploy SDK to notify about this step's success or failure (more info [here](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html)).
* `postTrafficHook`: (optional) validation Lambda function that runs after traffic shifting. It must use te CodeDeploy SDK to notify about this step's success or failure (more info [here](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html))
* `alarms`: (optional) list of CloudWatch alarms. If any of them is triggered duringt the deployment, the associated Lambda function will automatically roll back to the previous version.

## How it works

The plugin relies on the [AWS Lambda traffic shifting feature](https://docs.aws.amazon.com/lambda/latest/dg/lambda-traffic-shifting-using-aliases.html) to balance traffic between versions and [AWS CodeDeploy](https://docs.aws.amazon.com/lambda/latest/dg/automating-updates-to-serverless-apps.html) to automatically update its weight. It modifies the `CloudFormation` template generated by [Serverless](https://github.com/serverless/serverless), so that:

1. It creates a Lambda function Alias for each function with deployment settings.
2. It creates a CodeDeploy Application and adds a [CodeDeploy DeploymentGroup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-codedeploy-deploymentgroup.html) per Lambda function, according to the specified settings.
3. It modifies events that trigger Lambda functions, so that they invoke the newly created alias.

## Limitations

For now, the plugin only works with Lambda functions invoked by API Gateway or Stream based events (such as the triggered by Kinesis or DynamoDB Streams). More events will be added soon.

## License

ISC © [David García](https://github.com/davidgf)
