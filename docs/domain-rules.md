# Domain rules

This document details the encryption and the archive that [Architecture](architecture.md)
summarizes: how a note is encrypted and how a note is archived. The routes are in
[API endpoints](api.md), the workers in [Workers' responsibilities and flows](workers.md).

## How a note is encrypted

`getUserDataKey` in the encryption layer returns the caller's data key in plaintext, the key
that the users table stores wrapped by the notes key:

1. It reads `DEK(<user id>)` from Redis and returns the key on a hit
   (ADR 0002).
2. On a miss it reads the user's item from the users table and asks KMS to decrypt its
   `encryptedDEK`, then writes the plaintext key to Redis.
3. A user with no item gets a new 256-bit key from KMS `GenerateDataKey`; the plaintext goes to
   Redis and the encrypted copy to the users table.

With `ExistingRedisAddress` empty, the layer skips Redis and asks KMS on every call. The
`EnableUserDataKeysCache` parameter is declared but read by nothing; the Redis address alone
decides. Redis stays required all the same: the writing functions and the WebSocket
functions connect to it unconditionally for the connection ids.

The text is sealed with AES-256-GCM under a random 12-byte IV and stored as one binary value:
the IV, the ciphertext, then the 16-byte authentication tag. The title, ids and timestamps are
stored in plaintext.

## How a note is archived

[The stream processor](workers.md#notesstreamprocessor) turns every change into archive entries
in the same table:

- **Edit** — when a note that is not archived changes, the version before the change is written
  as a new item with the id `<id>-<old updatedAt>`, its title suffixed with the old date, marked
  `archived` and `unrestorable`. It can be read but not restored.
- **Delete** — a deleted note that was not archived is written back as `archived`, so it moves to
  the archive. Restoring it sets `archived` to false through `PUT /update/{id}`.

A change to an archived item creates no copy. The browser offers no delete for an archived note,
so the archive only grows.
