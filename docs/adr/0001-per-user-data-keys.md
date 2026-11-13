# 0001. Encrypt each user's notes with a data key of their own from KMS

- Status: Accepted
- Date: 2026-03-05

## Context

EncNotes stores the notes of many users in one DynamoDB table, and a user's notes must stay
unreadable to anyone holding another user's key. A KMS key per user would isolate users, but KMS
bills each key by the month, so thousands of users would mean thousands of keys to pay for and
manage. KMS can instead generate data keys under one key: it returns each new data key both in
plaintext and wrapped, and only KMS can unwrap the wrapped copy.

## Decision

- One KMS key, the notes key, serves the whole app.
- Each user gets one 256-bit data key from `GenerateDataKey` the first time a note of theirs is
  encrypted. Its wrapped copy is stored in a users table under the user's Cognito id; its
  plaintext is never stored in DynamoDB.
- A note's text is encrypted with the user's data key, AES-256-GCM with a random IV, inside the
  Lambda; the encryption code is a Lambda layer, kept in a repository of its own and included
  as a git submodule.

## Consequences

- A leaked data key exposes one user's notes, not everyone's.
- Every read or write needs the plaintext data key, which costs a KMS `Decrypt` call unless it is
  cached (ADR 0002).
- Only the text is encrypted; titles and timestamps stay in plaintext, so they can be listed and
  sorted without a key.
- Losing the notes key makes every note unreadable, so the stack keeps it when deleted.
