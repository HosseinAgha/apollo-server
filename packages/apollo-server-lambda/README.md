---
title: Lambda
description: Setting up Apollo Server with AWS Lambda
---

[![npm version](https://badge.fury.io/js/apollo-server-lambda.svg)](https://badge.fury.io/js/apollo-server-lambda) [![Build Status](https://circleci.com/gh/apollographql/apollo-server.svg?style=svg)](https://circleci.com/gh/apollographql/apollo-server) [![Coverage Status](https://coveralls.io/repos/github/apollographql/apollo-server/badge.svg?branch=master)](https://coveralls.io/github/apollographql/apollo-server?branch=master) [![Get on Slack](https://img.shields.io/badge/slack-join-orange.svg)](https://www.apollographql.com/#slack)

This is the AWS Lambda integration of GraphQL Server. Apollo Server is a community-maintained open-source GraphQL server that works with many Node.js HTTP server frameworks. [Read the docs](https://www.apollographql.com/docs/apollo-server/v2). [Read the CHANGELOG](https://github.com/apollographql/apollo-server/blob/master/CHANGELOG.md).

```sh
npm install apollo-server-lambda@rc
```

## Deploying with AWS Serverless Application Model (SAM)

To deploy the AWS Lambda function we must create a Cloudformation Template and a S3 bucket to store the artifact (zip of source code) and template. We will use the [AWS Command Line Interface](https://aws.amazon.com/cli/).

#### 1. Write the API handlers

```js
const { ApolloServer, gql } = require('apollo-server-lambda');

// Construct a schema, using GraphQL schema language
const typeDefs = gql`
  type Query {
    hello: String
  }
`;

// Provide resolver functions for your schema fields
const resolvers = {
  Query: {
    hello: () => 'Hello world!',
  },
};

const server = new ApolloServer({ typeDefs, resolvers });

exports.graphqlHandler = server.createHandler();
```

#### 2. Create an S3 bucket

The bucket name must be universally unique.

```bash
aws s3 mb s3://<bucket name>
```

#### 3. Create the Template

This will look for a file called graphql.js with the export `graphqlHandler`. It creates one API endpoints:

* `/graphql` (GET and POST)

In a file called `template.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  GraphQL:
    Type: AWS::Serverless::Function
    Properties:
      Handler: graphql.graphqlHandler
      Runtime: nodejs8.10
      Events:
        GetRequest:
          Type: Api
          Properties:
            Path: /graphql
            Method: get
        PostRequest:
          Type: Api
          Properties:
            Path: /graphql
            Method: post
```

#### 4. Package source code and dependencies

This will read and transform the template, created in previous step. Package and upload the artifact to the S3 bucket and generate another template for the deployment.

```sh
aws cloudformation package \
  --template-file template.yaml \
  --output-template-file serverless-output.yaml \
  --s3-bucket <bucket-name>
```

#### 5. Deploy the API

The will create the Lambda Function and API Gateway for GraphQL. We use the stack-name `prod` to mean production but any stack name can be used.

```
aws cloudformation deploy \
  --template-file serverless-output.yaml \
  --stack-name prod \
  --capabilities CAPABILITY_IAM
```

## Getting request info

To read information about the current request from the API Gateway event (HTTP headers, HTTP method, body, path, ...) or the current Lambda Context (Function Name, Function Version, awsRequestId, time remaning, ...) use the options function. This way they can be passed to your schema resolvers using the context option.

```js
const { ApolloServer, gql } = require('apollo-server-lambda');

// Construct a schema, using GraphQL schema language
const typeDefs = gql`
  type Query {
    hello: String
  }
`;

// Provide resolver functions for your schema fields
const resolvers = {
  Query: {
    hello: () => 'Hello world!',
  },
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ event, context }) => ({
    headers: event.headers,
    functionName: context.functionName,
    event,
    context,
  })
});

exports.graphqlHandler = server.createHandler();
```

## Modifying the Lambda Response (Enable CORS)

To enable CORS the response HTTP headers need to be modified. To accomplish this use `cors` options.

```js
const { ApolloServer, gql } = require('apollo-server-lambda');

// Construct a schema, using GraphQL schema language
const typeDefs = gql`
  type Query {
    hello: String
  }
`;

// Provide resolver functions for your schema fields
const resolvers = {
  Query: {
    hello: () => 'Hello world!',
  },
};

const server = new ApolloServer({ typeDefs, resolvers });

exports.graphqlHandler = server.createHandler({
  cors: {
    origin: '*',
    credentials: true,
  },
});
```

To enable CORS response for requests with credentials (cookies, http authentication) the allow origin header must equal the request origin and the allow credential header must be set to true.

```js
const { ApolloServer, gql } = require('apollo-server-lambda');

// Construct a schema, using GraphQL schema language
const typeDefs = gql`
  type Query {
    hello: String
  }
`;

// Provide resolver functions for your schema fields
const resolvers = {
  Query: {
    hello: () => 'Hello world!',
  },
};

const server = new ApolloServer({ typeDefs, resolvers });

exports.graphqlHandler = server.createHandler({
  cors: {
    origin: true,
    credentials: true,
  },
});
```

## Principles

GraphQL Server is built with the following principles in mind:

* **By the community, for the community**: GraphQL Server's development is driven by the needs of developers
* **Simplicity**: by keeping things simple, GraphQL Server is easier to use, easier to contribute to, and more secure
* **Performance**: GraphQL Server is well-tested and production-ready - no modifications needed

Anyone is welcome to contribute to GraphQL Server, just read [CONTRIBUTING.md](https://github.com/apollographql/apollo-server/blob/master/CONTRIBUTING.md), take a look at the [roadmap](https://github.com/apollographql/apollo-server/blob/master/ROADMAP.md) and make your first PR!