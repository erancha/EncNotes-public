# 0002. Cache plaintext data keys in Redis on the shared database host

- Status: Accepted
- Date: 2026-03-05

## Context

Every note request needs the caller's plaintext data key (ADR 0001). Getting it from the users
table and KMS on each request adds a KMS `Decrypt` call, billed per request, and its latency to
every list, add and update. The WebSocket functions also need a store that every Lambda can
reach for the mapping between users and their open connections. The account already ran Redis
on its shared database host, an EC2 instance outside this stack, while a managed cache cluster
bills by the hour whether used or not.

## Decision

- The encryption layer looks up `DEK(<user id>)` in Redis first. On a miss it unwraps the user's
  key through KMS, or generates one, and writes the plaintext to Redis.
- The same Redis holds each user's set of WebSocket connection ids and each connection's user.
- The stack takes the Redis address and password as parameters and creates no cache of its own.
  An empty address turns the key cache off, but the connection ids still need Redis, so the app
  does not run without one.

## Consequences

- A repeat request costs one Redis read instead of a DynamoDB read and a KMS call.
- Plaintext data keys sit in Redis with no expiry, so the Redis host is as sensitive as the keys
  themselves; whoever reads it can decrypt every cached user's notes.
- The app depends on a host this stack does not manage: when the host moves, the address in
  `scripts/lib/stack-params.sh` must follow, and its password lives in the gitignored
  `scripts/lib/secrets.sh`.
- The functions run outside a VPC and reach the host at its public address.
