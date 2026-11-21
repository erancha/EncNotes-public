# Design patterns

This document details the mechanisms that cut across the entry points
[Architecture](architecture.md) lists: where the shared code lives, how a reply reaches the
browser, how a setting reaches a function and the browser, where the functions run and what
wakes them; then which origin they answer.

**Contents:** [Shared code lives in layers](#shared-code-lives-in-layers) ·
[Replies go through a queue](#replies-go-through-a-queue) ·
[Settings come from the template and the stack's outputs](#settings-come-from-the-template-and-the-stacks-outputs) ·
[The functions run outside a VPC](#the-functions-run-outside-a-vpc) ·
[Nothing runs on a clock](#nothing-runs-on-a-clock) ·
[The allowed origin depends on the stack](#the-allowed-origin-depends-on-the-stack)

## Shared code lives in layers

The code several functions need lives in Lambda layers: `CorsHeadersLayer`
(`backend/layers/corsHeaders/`), the AWS SDK v3, ioredis and jsonwebtoken layers (zips in
`backend/layers/`, built from the `nodejs/` folders beside them), and `NotesEncryptionLayer`,
the git submodule at `backend/notes/layers/encryption/`. Lambda mounts them under `/opt/`, so a
function requires them by that path, for example `require('/opt/corsHeaders')`. How the zips and
the submodule are kept is in [Development & deployment](development.md#layers).

## Replies go through a queue

The note functions never push to a browser themselves. `AddNoteFunction`, `UpdateNoteFunction`
and `DeleteNoteFunction` queue each message to `MessagesToWebsocketsQueue`, and
`ExtractMessagesFunction`, the only function given the WebSocket API's address, pushes it to the
browser ([Workers' responsibilities and flows](workers.md#extractmessagesfunction)).

## Settings come from the template and the stack's outputs

The functions and the browser each read their settings from one place. `scripts/lib/stack-params.sh`
holds every parameter the stack is deployed with, the Redis password excepted, which comes from the
gitignored `scripts/lib/secrets.sh`. The template passes them to the functions as environment
variables: the table names, the KMS key id, `REDIS_ADDRESS`, `REDIS_PASSWORD`, the queue URL and
`ALLOWED_ORIGIN`.

The browser reads the stack's outputs from `appConfig.json`, which `lib/app-config.sh` writes:
the app fetches the file from the same origin at startup for its API URLs and Cognito ids.

The notes and users tables and the notes key belong to no stack: `stack-params.sh` passes their
names to both branch stacks, so both read and write the same notes. The template can create
each of them instead, when its parameter is empty.

## The functions run outside a VPC

Redis runs on the account's shared database host, an EC2 instance outside this stack, reached
with a password. The networking parameters of the template are left empty, so the functions run
outside any VPC.

## Nothing runs on a clock

No schedule wakes any function of the stack: each one runs on a request, a queue message or a
stream record.

## The allowed origin depends on the stack

The functions answer with the origin of `ALLOWED_ORIGIN` in their CORS headers: the CloudFront
address on the `en` stack, any origin on `en-fb`. The REST API's own preflight and error
responses allow any origin.
