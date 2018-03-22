![serverless components logo](https://s3.amazonaws.com/assets.github.serverless/serverless-components-readme-2.gif)

## Overview

### Easy

A Serverless Component can package _cloud/SaaS services_, _logic_ & _automation_ into a simple building block you can use to build applications more easily than ever.

### Composable

Serverless Components can be combined & nested. Reuse their functionality to build applications faster. Combine and nest them to make higher-order components, like features or entire applications.

### Open

Serverless Components are 100% open-source & vendor-agnostic. You choose the services that best solve your problem, instead of being limited and locked into one platform.

### Serverless

Serverless Components can deploy anything, but they're biased toward SaaS & cloud infrastructure with "serverless" qualities (auto-scaling, pay-per-use, zero-administration), so you can deliver apps with the lowest total cost & overhead.

![serverless components overview](https://s3.amazonaws.com/assets.github.serverless/serverless-components-overview-2.gif)

• [Join the discussion on Slack - Serverless Contrib](https://join.slack.com/t/serverless-contrib/shared_invite/enQtMjI5NzY1ODM2MTc3LTRhNGRlMWQyZTdhY2Y1OGIzNTdhZWUyYTVjYzA4ZDNhMDcyOTVjZWU1YWVhODExMTgzMTVmYWY0MzdjZmRjODI) •

## Table of contents

* [Getting started](#getting-started)
* [Current limitations](#current-limitations)
* [Concepts](#concepts)
  * [Components](#components)
  * [Input types, Inputs & Outputs](#input-types-inputs-outputs)
  * [State](#state)
  * [Variables](#variables)
  * [Graph](#graph)
  * [Custom commands](#custom-commands)
  * [Registry](#registry)
* [Creating components](#creating-components)
  * [Basic setup](#basic-setup)
  * [`serverless.yml`](#serverless.yml)
  * [`index.js`](#index.js)
  * [Testing](#testing)
* [Docs](#docs)
  * [aws-apigateway](#aws-apigateway)
  * [aws-cloudfront](#aws-cloudfront)
  * [aws-dynamodb](registry/aws-dynamodb/README.md)
  * [aws-iam-policy](#aws-iam-policy)
  * [aws-iam-role](#aws-iam-role)
  * [aws-lambda](#aws-lambda)
  * [aws-route53](#aws-route53)
  * [aws-s3-bucket](#aws-s3-bucket)
  * [eventgateway](#eventgateway)
  * [github](#github)
  * [github-webhook-receiver](#github-webhook-receiver)
  * [rest-api](#rest-api)
  * [s3-dirloader](#s3-dirloader)
  * [s3-downloader](#s3-downloader)
  * [s3-policy](#s3-policy)
  * [s3-sync](#s3-sync)
  * [s3-uploader](#s3-uploader)
  * [s3-website-config](#s3-website-config)
  * [static-website](#static-website)


## Getting started

**Note:** Make sure you have Node.js 8+ and npm installed on your machine
Also please do join the _Components_ channel on our public [Serverless-Contrib Slack](https://serverless-contrib.slack.com/messages/C9U3RA55M) to continue the conversation.

1. Setup
1. `npm install --global serverless-components`
1. Setup the environment variables
   * `export AWS_ACCESS_KEY_ID=my_access_key_id`
   * `export AWS_SECRET_ACCESS_KEY=my_secret_access_key`

### Running Locally

Run commands with

```
components [Command]
```

## Current limitations

The following is a list with some limitations one should be aware of when using this project.
**NOTE:** We're currently working on fixes for such issues and will announce them in our release notes / changelogs.

### `us-east-1` only

Right now the only supported region is `us-east-1`

### No rollback support

Rolling back your application into the previous, stable state is currently not supported.

However the framework ensures that your state file always reflects the correct state of your infrastructure setup (even if something goes wrong during deployment / removal).

## Concepts

### Components


A component is the smallest unit of abstraction for your infrastructure. It can be a single small piece like an IAM role, or a larger piece that includes other small pieces, like [`github-webhook-receiver`](#github-webhook-receiver), which includes `aws-lambda` (which itself includes `aws-iam-role`), `aws-apigateway` (which also includes `aws-iam-role`), `aws-dynamodb`, and `github`. So components can be composed with each other in a component dependency graph to build larger components.

You define a component using two files: `serverless.yml` for config, and `index.js` for the provisioning logic.

The `index.js` file exports multiple functions that take two arguments: `inputs` and `context`. Each exported function name reflects the CLI command which will invoke it (the `deploy` function will be executed when one runs `components deploy`).

These two files look something like this:

**serverless.yml**

```yml
type: my-component

inputTypes: # type descriptions for inputs that my-component expects to receive
  name:
    type: string
    required: true
  age:
    type: number
    required: false
    default: 47
```

**index.js**

```js
const deploy = (inputs, context) => {
  // provisioning logic goes here
}

module.exports = {
  deploy // this is the command name which is exposed via the CLI
}
```

However, this `index.js` file is optional, since your component can just be a composition of other smaller components without provisioning logic on its own.

### Input types, Inputs & Outputs

#### Input types


Input types are the description of the inputs your component receives. You supply those `inputTypes` in the component's `serverless.yml` file:

```yml
type: child-component

inputTypes:
  name:
    type: string
    required: true
    default: John
```


Or, if the component is being used as a child of another parent component, the parent will supply `inputs` and they can overwrite the defaults that are defined at the child level:

```yml
type: parent-component

components:
  myChild:
    type: child-component
    inputs:
      name: Jane  # This overrides the default of "John" from the inputType
```

#### Inputs

Inputs are the configuration that are supplied to your components logic by the user. You define these inputs in the `serverless.yml` file where the component is used:

```yml
type: my-application

components:
  myFunction:
    type: aws-lambda
    inputs:
      memory: 512
      timeout: 300
```

Given this `serverless.yml` you would deploy a `aws-lambda` function with a memory size of 512 and timeout of 300.

#### Outputs

Your provisioning logic, or the `deploy` method of your `index.js` file, can optionally return an `outputs` object. This output can be referenced in `serverless.yml` as inputs to another component.

For example, the lambda component's deploy method returns outputs that look like this...

**index.js**

```js
const deploy = (inputs, context) => {
  // lambda provisioning logic

  // return outputs
  return {
    arn: res.FunctionArn
  }
}

module.exports = {
  deploy
}
```

These outputs can then be referenced by other components. In this example we reference the function arn and pass it in to the `aws-apigateway` component to set up a handler for the route.

```yml
type: my-application

components:
  myFunction:
    type: aws-lambda
    inputs:
      handler: code.handler
  myEndpoint:
    type: aws-apigateway
    inputs:
      routes:
        /github/webhook:
          post:
            lambdaArn: ${myFunction.arn}
```

### State

State can be accessed via the `context` object and represents a historical snapshot of what happened the last time you ran a command such as `deploy`, `remove`, etc.

The provisioning logic can use this state object and compare it with the current inputs to make decisions around whether to run deploy, update or remove.

The operation that will be fired depends on the inputs and how the provider works. Change in some inputs for some provider could trigger a create / remove while other inputs might trigger an update. It's up to the component to decide.

Here's an example demonstrating how a lambda component decides what needs to be done based on the inputs and state:

```js
const deploy = (inputs, context) => {
let outputs;
  if (inputs.name && !context.state.name) {
    console.log(`Creating Lambda: ${inputs.name}`);
    outputs = await create(inputs);
  } else if (context.state.name && !inputs.name) {
    console.log(`Removing Lambda: ${context.state.name}`);
    outputs = await remove(context.state.name);
  } else if (inputs.name !== context.state.name) {
    console.log(`Removing Lambda: ${context.state.name}`);
    await remove(context.state.name);
    console.log(`Creating Lambda: ${inputs.name}`);
    outputs = await create(inputs);
  } else {
    console.log(`Updating Lambda: ${inputs.name}`);
    outputs = await update(inputs);
  }
  return outputs;
}

module.exports = {
  deploy
}
```

### Variables

The framework supports variables from the following sources:

* **Environment Variables:** for example, `${env.GITHUB_TOKEN}`
* **Output:** for example: `${myEndpoint.url}`, where `myEndpoint` is the component alias as defined in `serverless.yml`, and `url` is a property in the outputs object that is returned from the `myEndpoint` provisioning function.

### Graph

Once you start composing components together with multiple levels of nesting, and all of these components depend on one another with variable references, you then end up with a graph of components.

Internally, the framework constructs this dependency graph by analyzing the entire component structure and their variable references. With this dependency graph the framework is able to provision the required components in parallel whenever they either don't depend on each other, or are waiting on other components that haven't been provisioned yet.

The component author / user doesn't have to worry about this graph at all, they just use variables to reference the outputs which should be used and it will just work.

### Custom Commands

Other than the built in `deploy` and `remove` commands, you can also include custom commands to add extra management capability for your component lifecycle. This is achieved by adding the corresponding function to the `index.js` file, just like the other functions in `index.js`.

The function receives `inputs` and `context` as parameters.

```js
const deploy = (inputs, context) => {
  // some provisioning logic
}

const test = (inputs, context) => {
  console.log('Testing the components functionality...')
}

module.exports = {
  deploy,
  test // make the function accessible from the CLI
}
```

### Registry

The ["Serverless Registry"](./registry) is a core part in the components implementation as it makes it possible to discover, publish and share existing components. For now, `serverless-components` ships with a number of built-in components that are usable by type name.

The registry is not only limited to serving components. Since components are functions, it's possible to wrap existing business logic into functions and publish them to the registry as well.

Looking into the future, it will even be possible to serve functions which are written in different languages through the registry.

## Creating components

Here is a quick guide to help you kick-start your component development.

**Note:** Make sure to re-visit the [core concepts](#concepts) above before you jump right into the component implementation.

### Basic setup

In this guide we'll build a simple `greeter` component which will greet us with a custom message when we run the `deploy`, `greet` or `remove` commands.

First we need to create a dedicated directory for our component. This directory will include all the necessary files for our component, like its `serverless.yml` file, the `index.js` file (which includes the components logic), and files such as `package.json` to define its dependencies.

Go ahead and create a `greeter` directory in the "Serverless Registry" directory located at [`registry`](./registry).

### `serverless.yml`

Let's start by describing our components interface. We define the interface with the help of a `serverless.yml` file. Create this file in the components directory and paste in the following content:

```yml
type: greeter

inputTypes:
  firstName:
    type: string
    required: true
  lastName:
    type: string
    required: true
```

Let's take a closer look at the code we've just pasted. At first we define the `type` (think of it as an identifier or name) of the component. In our case the component is called `greeter`.

Next up we need to declare the `inputTypes` our component has. `inputTypes` define the shape our inputs take and are accessible from within the components logic. In our case we expect a `firstName` and a `lastName`.

That's it for the component definition. Let's move on to the implementation of its logic.

### `index.js`

The components logic is implemented with the help of an `index.js` file which is located in the root of the components directory. Go ahead and create an empty `index.js` file in the components root directory.

Then we'll implement the logic for the `deploy`, `greet` and `remove` commands. We do this by adding the respective functions into the file and exporting them so that the framework CLI can pick them up (_Remember:_ only the exported functions are accessible via CLI commands).

Just paste the following code in the `index.js` file:

```js
// "private" functions
function greetWithFullName(inputs, context) {
  context.log(`Hello ${inputs.firstName} ${inputs.lastName}!`)
}

// "public" functions
function deploy(inputs, context) {
  greetWithFullName(inputs, context)

  if (context.state.deployedAt) {
    context.log(`Last deployment: ${context.state.deployedAt}...`)
  }

  const deployedAt = new Date().toISOString()

  const updatedState = {
    ...context.state,
    ...inputs,
    deployedAt
  }
  context.saveState(updatedState)

  return updatedState
}

function greet(inputs, context) {
  context.log(`Hola ${inputs.firstName} ${inputs.lastName}!`)
}

function remove(inputs, context) {
  greetWithFullName(inputs, context)
  context.log('Removing...')
  context.saveState()
}

module.exports = {
  deploy,
  greet,
  remove
}
```

Let's take a closer look at the implementation.

Right at the top we've defined a "helper" function we use to reduce code duplication (this function is not exported at the bottom and can therefore only be used internally). This `greetWithFullName` function gets `inputs` and `context`, and then logs a message which greets the user with his full name. Note that we're using the `log` function which is available at the `context` object instead of the native `console.log` function. The `context` object has other, very helpful functions and data attached to it.

Next up we've defined the `deploy` function. This function is executed every time the user runs a `deploy` command since we've exported it at the bottom of the file. At first we re-use our `greetWithFullName` function to greet our user. Then we check the state to see if we've already deployed it previously. If this is the case we log out the timestamp of the last deployment. After that we get the current time and store it in an object which includes the `state`, the `inputs` and the new `deployedAt` timestamp. This object reflects our current state, which we now store. After that we return the object as outputs.

The `greet` function is a custom `command` function we use to extend the CLI's capabilities. Since we've exported it at the bottom of the file it'll be executed every time someone runs the `greet` command. The functionality is pretty straightforward. We just log out a different greeting using the `context.log` method and the `inputs`.

The last function we've defined in our components implementation is the `remove` function. Remove is also accessible from the CLI because we export it at the bottom of the file. The functions code is also pretty easy to understand. At first we greet our user with the `greetWithFullName` helper function. Next up we log a message that the removal was triggered and store an empty state (meaning that there's no more state information available). That's it.

### Testing

Let's test our component!

First of all let's create a new example application which uses our `greeter` component. `cd` into the `examples` directory by running:

```sh
cd examples
```

Create a new directory named `test` which has one `serverless.yml` file with the following content:

```yml
type: my-application

components:
  myGreeter:
    type: greeter
    inputs:
      firstName: John
      lastName: ${env.LAST_NAME}
```

If we take a closer look at the `serverless.yml` file we can see that our `lastName` config value depends on a variable called `LAST_NAME` that is fetched from the local environment. This means that we need to export this variable so that the framework can pick it up and pass it down into our `inputs`:

```sh
export LAST_NAME=Doe
```

That's it. Let's take it for a spin. Run the following commands to test the components logic:

```
../../bin/components deploy

../../bin/components deploy

../../bin/components greet

../../bin/components remove
```

Congratulations! You've successfully created your first Serverless component!

Want to learn more? Make sure to take a look at all the different component implementations in the ["Serverless Registry"](./registry)!

## Docs

### CLI Usage

#### Deployment

To deploy your app, run

```
components deploy
```

#### Updating

* Change some components inputs
* Then run

```
components deploy
```

#### Removal

To remove your app, run

```
components remove
```

### Component Usage

#### `aws-apigateway`

Creates / Removes an API endpoint which is exposed via AWS API Gateway and connects directly to an AWS Lambda function.

##### Inputs

| Name      | Description                                                      | Type   |
| --------- | ---------------------------------------------------------------- | ------ |
| `name`    | The API name                                                     | String |
| `roleArn` | The AWS IAM Role `arn` which should be used for this API Gateway | String |
| `routes`  | The routes definitions for this API                              | Object |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myApiGateway:
    type: aws-apigateway
    inputs:
      name: my-apigateway
      roleArn: arn:aws:iam::XXXXX:role/some-api-gateway-role
      routes:
        /hello:
          post:
            handler: ${helloPostLambdaComponent}
            cors: true
          get:
            handler: ${helloGetLambdaComponent}
        /hello/world:
          delete:
            handler: ${helloWorldLambdaComponent}
```


#### `aws-cloudfront`

##### Inputs

| Name                    | Description                               | Type    |
| ----------------------- | ----------------------------------------- | ------- |
| `name`                  | The name                                  | String  |
| `defaultRootObject`     | The default root object                   | String  |
| `originDomain`          | The origin domain                         | String  |
| `aliasDomain`           | The alias domain                          | String  |
| `distributionEnabled`   | Whether the distribution is enabled       | Boolean |
| `loggingEnabled`        | Whether loggig is enabled                 | Boolean |
| `loggingBucket`         | The logging bucket                        | String  |
| `loggingIncludeCookies` | Whether cookies are oncluded when logging | Boolean |
| `loggingPrefix`         | The logging prefix                        | String  |
| `priceClass`            | The price class                           | String  |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myCloudFront:
    type: aws-cloudfront
    inputs:
      name: cf-bucketname  # sent to CallerReference
      defaultRootObject: index.html
      originDomain: bucketname.s3.amazonaws.com  # later accept a list
      aliasDomain: www.example.com # later accept a list
      distributionEnabled: false
      loggingEnabled: false
      loggingBucket: logbucket
      loggingIncludeCookies: false
      loggingPrefix: cf-bucketname-
      priceClass: PriceClass_All # PriceClass_100 | PriceClass_200 | PriceClass_All
```


#### `aws-dynamodb`

Creates / Removes an AWS DynamoDB table.

##### Inputs

| Name             | Description                                                                                              | Type   |
| ---------------- | -------------------------------------------------------------------------------------------------------- | ------ |
| `region`         | The region in which this table should be created                                                         | String |
| `deletionPolicy` | The deletion policy this table should follow                                                             | String |
| `properties`     | Property definitions to describe the tables config in detail (See info below for in-depth documentation) | Object |

**Note:** You can find the full range of configurable parameters in the [AWS docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-dynamodb-table.html)

```yml
type: my-application

components:
  myDynamoDBTable:
    type: aws-dynamodb
    inputs:
      region: us-east-1
      deletionPolicy: Delete
      properties:
        tableName: my-table
        attributeDefinitions:
          - attributeName: id
            attributeType: S
        keySchema:
          - attributeName: id
            keyType: HASH
        globalSecondaryIndexes: null
        localSecondaryIndexes: null
        provisionedThroughput:
          readCapacityUnits: 1
          writeCapacityUnits: 1
        streamSpecification: null
        tags:
          - tag-1
          - tag-2
        timeToLiveSpecification:
          attributeName: ttl
          enabled: true
        sSESpecification:
          sSEEnabled: false
```

##### Commands

* `components deploy`
* `components remove`

##### Example

[See Default inputs](./registry/aws-dynamodb)



#### `aws-iam-policy`

##### Inputs

| Name         | Description     | Type   |
| ------------ | --------------- | ------ |
| `name`       | The name        | String |
| `bucketName` | The bucket name | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myIamPolicy:
    type: aws-iam-policy
    inputs:
      name: default-role-name-goes-here
      bucketName: some-unique-bucket-name
```


#### `aws-iam-role`

Creates / Removes an AWS IAM role.

##### Inputs

| Name      | Description                                 | Type   |
| --------- | ------------------------------------------- | ------ |
| `name`    | The name for the IAM role                   | String |
| `service` | The service this role should be created for | String |
| `policy`  | The AWS policy to apply to this role        | Object |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myIamRole:
    type: aws-iam-role
    inputs:
      name: role-name
      service: lambda.amazonaws.com
      policy:
        arn: some-policy-arn
```


#### `aws-lambda`

Creates / Removes an AWS Lambda function.

##### Inputs

| Name          | Description                          | Type   |
| ------------- | ------------------------------------ | ------ |
| `name`        | The functions name                   | String |
| `memory`      | The functions memory size            | Number |
| `timeout`     | The functions timeout                | Number |
| `description` | The functions description            | String |
| `handler`     | The functions handler                | String |
| `role`        | The role `arn` the lambda should use | String |
| `env`         | An object of env vars set at runtime | Object |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myLambda:
    type: aws-lambda
    inputs:
      name: some-lambda-function
      memory: 128
      timeout: 10
      description: dome description
      handler: code.handler
      role:
        arn: arn:aws:iam::XXXXX:role/some-lambda-role
```


#### `aws-route53`

##### Inputs

| Name          | Description                    | Type    |
| ------------- | ------------------------------ | ------- |
| `name`        | The name                       | String  |
| `domainName`  | The domain name                | String  |
| `dnsName`     | The DNS name                   | String  |
| `privateZone` | Whether this is a private zone | Boolean |
| `vpcId`       | The VPCs Id                    | String  |
| `vpcRegion`   | The VPSs region                | String  |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myRoute53:
    type: aws-route53
    inputs:
      name: mydefaulthostedzone  # sent to CallerReference
      domainName: www.defaultdomain.com
      dnsName: d111111abcdef8.cloudfront.net
      privateZone: false
      vpcId: my-existing-vpc-id
      vpcRegion: mydefaultvpcregion
```


#### `aws-s3-bucket`

Creates / Removes an AWS S3 bucket.

##### Inputs

| Name   | Description     | Type   |
| ------ | --------------- | ------ |
| `name` | The bucket name | String |
| `forceUniqueBucketName` | Force unique postFix on bucket name to avoid bucket name collisions | Boolean |


##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myS3:
    type: aws-s3-bucket
    inputs:
      name: default-bucket-name
```


#### `eventgateway`

Creates / Removes an Event Gateway function registration and corresponding subscription.

##### Inputs

| Name                 | Description                                                                    | Type    |
| -------------------- | ------------------------------------------------------------------------------ | ------- |
| `event`              | `http` or a freeform event like `user.created`                                 | String  |
| `path`               | The event path                                                                 | String  |
| `method`             | **Optional** The HTTP method (if the `http` event is used)                     | String  |
| `cors`               | **Optional** If CORS support should be setup                                   | Boolean |
| `space`              | The Event Gateway space which should be used                                   | String  |
| `eventGatewayApiKey` | The Event Gateway API Key which is used to manage configuration on your behalf | String  |
| `lambdaArn`          | The functions runtime                                                          | String  |

##### Commands

* `components deploy`
* `components remove`
* `components info`

##### Example

```yml
type: my-application

components:
  myEventGateway:
    type: eventgateway
    inputs:
      event: http # or any freeform event like "user.created"
      path: some-path
      method: POST # optional
      cors: true # optional
      space: some-space
      eventGatewayApiKey: s0m33v3ntg4t3w4y4p1k3y
      lambdaArn: arn:aws:lambda:us-east-1:XXXXX:function:some-lambda-function
```


#### `github`

Creates / Removes a GitHub Webhook.

##### Inputs

| Name    | Description                                                         | Type   |
| ------- | ------------------------------------------------------------------- | ------ |
| `token` | The GitHub token which is used to create the Webhook on your behalf | String |
| `owner` | The repository owner                                                | String |
| `repo`  | The name of the GitHub repository                                   | String |
| `url`   | The Webhook URL                                                     | String |
| `event` | The Webhooks event                                                  | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myGitHub:
    type: github
    inputs:
      token: ${GITHUB_TOKEN}
      owner: jdoe
      repo: some-repository
      url: https://some-endpoint.com
      event: push
```


#### `github-webhook-receiver`

Creates / Tests / Removes a GitHub Webhook receiver.

##### Inputs

| Name  | Description     | Type   |
| ----- | --------------- | ------ |
| `url` | The Webhook URL | String |

##### Commands

* `components deploy`
* `components test`
* `components remove`

##### Example

**Note:** This example re-uses other components.

```yml
type: my-application

components:
  myGitHubWebhookReceiver:
    type: github-webhook-receiver
    inputs:
      url: https://my.endpoint
```


#### `rest-api`

Creates / Removes a RESTful API. Uses the specified gateway component behind the scenes.

##### Inputs

| Name                 | Description                                                                | Type   |
| -------------------- | -------------------------------------------------------------------------- | ------ |
| `gateway`            | The component name of the gateway which should be used (e.g. `apigateway`) | String |
| `eventGatewayApiKey` | **Optional** The Event Gateway API Key                                     | String |
| `space`              | **Optional** The space used by the Event Gateway                           | String |
| `name`               | The APIs name                                                              | String |
| `routes`             | The routes this RESTful API should expose                                  | Object |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myRestApi:
    type: rest-api
    inputs:
      gateway: eventgateway
      eventGatewayApiKey: s0m33v3ntg4t3w4y4p1k3y # optional
      space: some-space # optional
      name: some-rest-api
      routes:
        /products: # routes begin with a slash
          post: # HTTP method names are used to attach handlers
            function: { 'arn': 'arn:aws:lambda:us-east-1:XXXXX:function:products-create' } # value can be a direct component reference or an object containing the Lambda ARN

          # sub-routes can be declared hierarchically
          /{id}: # path parameters use curly braces
            get:
              function: { 'arn': 'arn:aws:lambda:us-east-1:XXXXX:function:products-get' }
              cors: true # CORS can be allowed with this flag

        # multi-segment routes can be declared all at once
        /catalog/{...categories}: # catch-all path parameters use ellipses
          get:
            function: { 'arn': 'arn:aws:lambda:us-east-1:XXXXX:function:products-list' }
            cors: true
```


#### `s3-dirloader`

##### Inputs

| Name          | Description      | Type   |
| ------------- | ---------------- | ------ |
| `contentPath` | The content path | String |
| `bucketName`  | The bucket name  | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myS3DirLoader:
    type: s3-dirloader
    inputs:
      contentPath: ./site
      bucketName: content-bucket
```


#### `s3-downloader`

Creates / Removes a component which makes it possible to listen to a "file uploaded" event and log the event data in a DynamoDB table.

##### Inputs

| Name   | Description                 | Type   |
| ------ | --------------------------- | ------ |
| `name` | The name for this component | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

**Note:** This example re-uses other components.

```yml
type: my-application

components:
  myS3Downloader:
    type: s3-downloader
    inputs:
      name: some-name
```


#### `s3-policy`

##### Inputs

| Name         | Description     | Type   |
| ------------ | --------------- | ------ |
| `bucketName` | The bucket name | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myS3Policy:
    type: s3-policy
    inputs:
      bucketName: some-unique-bucket-name
```


#### `s3-sync`

##### Inputs

| Name          | Description      | Type   |
| ------------- | ---------------- | ------ |
| `contentPath` | The content path | String |
| `bucketName`  | The bucket name  | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myS3Sync:
    type: s3-sync
    inputs:
      contentPath: ./site
      bucketName: content-bucket
```


#### `s3-uploader`

Creates / Removes a component which makes it possible to upload files via the Event Gateway and store them in an AWS S3 bucket.

##### Inputs

| Name   | Description                 | Type   |
| ------ | --------------------------- | ------ |
| `name` | The name for this component | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myS3Uploader:
    type: s3-uploader
    inputs:
      name: some-name
```


#### `s3-website-config`

##### Inputs

| Name                 | Description                  | Type   |
| -------------------- | ---------------------------- | ------ |
| `rootBucketName`     | The root buckets name        | String |
| `indexDocument`      | The index document           | String |
| `errorDocument`      | The error document           | String |
| `redirectBucketName` | The redirect bucket name     | String |
| `redirectToHostName` | The redirect to the hostname | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

```yml
type: my-application

components:
  myWebsiteConfig:
    type: s3-website-config
    inputs:
      rootBucketName: default.com
      indexDocument: index.html
      errorDocument: error.html
      redirectBucketName: www.default.com
      redirectToHostName: default.com
```


#### `static-website`

##### Inputs

| Name            | Description        | Type   |
| --------------- | ------------------ | ------ |
| `name`          | The name           | String |
| `contentPath`   | The content path   | String |
| `contentIndex`  | The content index  | String |
| `contentError`  | The content error  | String |
| `hostingRegion` | The hosting region | String |
| `hostingDomain` | The hosting domain | String |
| `aliasDomain`   | The alias domain   | String |

##### Commands

* `components deploy`
* `components remove`

##### Example

**Note:** This example re-uses other components.

```yml
type: my-application

components:
  myStaticWeb:
    type: static-website
    inputs:
      name: familysite
      contentPath: ./site
      contentIndex: index.html
      contentError: error.html
      hostingRegion: us-east-1
      hostingDomain: myfamily.com
      aliasDomain: www.myfamily.com
```
