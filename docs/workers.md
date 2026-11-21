# Workers' responsibilities and flows

The workers are `NotesStreamProcessor` and `ExtractMessagesFunction`; this document details the
steps, limits and retries of each, behind the entry points that
[Architecture](architecture.md#the-entry-points) summarizes. A worker is a function no request
waits on: a table's stream or a queue message wakes it. The routes that feed them are in
[API endpoints](api.md).

## `NotesStreamProcessor`

`NotesStreamProcessor` (`backend/notes/stream/`) reads the notes table's stream, which carries
each item's image before the change, and keeps the archive
([How a note is archived](domain-rules.md#how-a-note-is-archived)).

## `ExtractMessagesFunction`

`MessagesToWebsocketsQueue`, a FIFO queue with content-based deduplication, carries the notices
to `ExtractMessagesFunction` (`backend/websocket/extractMessages/`). Every notice goes to one
message group, so notices are delivered in the order they were sent. A notice is the connection
id, a `refresh` command and a line of text naming the note.

The function is triggered by the queue in batches of up to 10, and posts each notice to its
connection through the API Gateway management API. A connection that has gone is logged and left
in Redis.
