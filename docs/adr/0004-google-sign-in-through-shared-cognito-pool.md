# 0004. Sign in with Google through a Cognito user pool another stack owns

- Status: Accepted
- Date: 2026-03-05

## Context

Every note belongs to one user, and the per-user data key (ADR 0001) is looked up by a user id,
so each request must carry an identity the backend can trust. The account already ran a Cognito
user pool, with a hosted sign-in domain and Google configured as an identity provider, for
another app. A pool of its own would mean a second domain to register as a Google OAuth redirect
and a second set of users to manage.

## Decision

- The stack takes the existing pool and its domain as the `ExistingUserPoolId` and
  `ExistingCognitoDomain` parameters and creates only an app client, with the authorization-code
  flow and Google as its only identity provider.
- The browser signs in through Amplify's redirect to the pool's hosted page and sends the ID
  token as a bearer token. The REST API's Cognito authorizer checks it on every route, and the
  functions take the user id from its `sub` claim.
- The WebSocket API has no authorizer; the browser passes the ID token in the `token` query
  parameter, and the connect function reads `sub` from it to record the connection.

## Consequences

- No passwords are stored or checked by this app; a user is whoever Google says they are.
- Users of the other app share the pool and can sign in here; each still sees only their own
  notes, since every query is keyed by `sub`.
- The pool's lifetime is not this stack's: deleting the stack keeps every user.
- The connect function decodes the token without checking its signature, so a forged token can
  register a connection under another user's id and receive that user's change notices.
