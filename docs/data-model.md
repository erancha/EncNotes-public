# Data model

This document details the stores that [Architecture](architecture.md#storage) summarizes: the
two DynamoDB tables with their keys and index, the frontend bucket and the keys Redis holds.

## DynamoDB

Both tables bill per request. A table or key the stack creates is kept when the stack is
deleted; the deployed ones belong to no stack
([Settings come from the template and the stack's outputs](design-patterns.md#settings-come-from-the-template-and-the-stacks-outputs)).

- **notes table** — key `id`; `userId`, `title`, `content` (binary), `createdAt`, `updatedAt`,
  optional `archived` and `unrestorable`; index `UserIdUpdatedIndex` on `userId` and
  `updatedAt`; stream of old images; point-in-time recovery.
- **users table** — key `userId`; `encryptedDEK` (binary), the user's data key wrapped by the
  notes key.

## S3

The frontend bucket holds the React build, with website hosting and public read access
(`S3PublicAccess=true` in `stack-params.sh`). CloudFront reaches it through the bucket's website
endpoint over HTTP and redirects viewers to HTTPS.

## Redis

Redis holds three kinds of key, named by the user id or the connection id:

- `DEK(<user id>)` — a string: the user's plaintext data key, with no expiry.
- `connections(<user id>)` — a set: the user's open WebSocket connection ids.
- `userId(<connection id>)` — a string: the user a connection belongs to.
