# 0003. Serverless on AWS, defined in one SAM template

- Status: Accepted
- Date: 2026-03-05

## Context

A notes app has work only while a user has it open; between requests there is nothing to do, so
a server running all month would be paid for while idle. The app needs authenticated REST calls,
push notices to a user's open sessions, and a static frontend, all of which AWS offers as managed
services billed per use.

## Decision

- Node.js Lambdas behind an API Gateway REST API with a Cognito authorizer for notes, and an
  API Gateway WebSocket API for push notices, with an SQS FIFO queue between the note writes and
  the function that posts the notices.
- DynamoDB tables billed per request; a DynamoDB stream drives the archive of old versions.
- A React app in S3 behind CloudFront.
- The whole stack in one AWS SAM template, `scripts/template.yaml`, with the shared libraries as
  Lambda layers, built and deployed by `sam build` and `sam deploy`.

## Consequences

- An idle month costs little beyond storage and the KMS key.
- A note write returns before its notices are posted; the queue decouples the two.
- SAM shortens function and layer declarations, but anything it does not model (the WebSocket
  API, its routes and permissions) is written as plain CloudFormation in the same template.
- The tables and the notes key outlive the stack, so data survives a stack's deletion.
