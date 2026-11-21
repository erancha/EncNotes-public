# API endpoints

This document details every route behind the entry points that
[Architecture](architecture.md#the-entry-points) summarizes: who may call it, what it takes and
answers, and its limits. The browser reaches the app through a WebSocket API and a REST API, each
on a stage named by the branch's stack
([Deployment scripts](development.md#deployment-scripts-scripts)). The notices a write queues are
delivered by the workers ([Workers' responsibilities and flows](workers.md)).

## WebSocket API

The browser connects to the WebSocket API (`WebSocketApi`) with its ID token in the `token`
query parameter.

- `$connect` goes to `ConnectFunction` (`backend/websocket/connect/`), which reads the user id from
  the token's `sub` claim and records the connection in Redis.
- `$disconnect` goes to `DisconnectFunction`, which removes the connection.

No other route exists. The stage writes access logs to a log group of the stack and needs the
account's API Gateway logging role
([What must exist outside the stack](development.md#what-must-exist-outside-the-stack)).

## REST API

The REST API (`RESTApi`) has a Cognito authorizer that checks the token on every route. The
routes, one function each, under `backend/notes/`:

- `GET /list` (`ListNotesFunction`) — queries the caller's notes by the `UserIdUpdatedIndex`
  index, newest first, in one query with no paging, and decrypts each text. With a
  `searchTerm` it filters the decrypted notes in the function by title, text or both,
  optionally case-sensitive. It returns archived notes too; the browser splits active from
  archived.
- `POST /add` (`AddNoteFunction`) — stores a note under a new random id, its text encrypted.
- `PUT /update/{id}` (`UpdateNoteFunction`) — a body with a title replaces the title and the
  re-encrypted text; a body without one sets `archived`, which is how a note is restored.
- `DELETE /delete/{id}` (`DeleteNoteFunction`) — deletes the item; the stream processor puts
  it back as archived.

After its write, each writing function (add, update, delete) reads the caller's connection ids
from Redis and queues one notice per connection.
