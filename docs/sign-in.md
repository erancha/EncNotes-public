# Sign-in

A user signs in to EncNotes with a Google account, and there is no other way in; the account's
user id is the key to its notes and to its data key. This page details the Cognito entry of
[Architecture](architecture.md#the-entry-points): how a sign-in reaches the APIs, what the stack
adds to the user pool it signs in through, and how to set it up in a new AWS account. The pool
is not this repo's: the `vsdb-cognito` stack of the vpc-subnets-sgs repo owns it, together with
its Google client, and several apps of the account sign in through it
(ADR 0004).

```mermaid
sequenceDiagram
    participant B as Browser app
    participant C as Cognito hosted domain
    participant G as Google
    participant A as REST and WebSocket APIs
    B->>C: /oauth2/authorize with the app client id, identity provider Google
    C->>G: Google sign-in
    G->>C: /oauth2/idpresponse
    C->>B: back to the app address with an authorization code
    B->>C: /oauth2/token exchanges the code for tokens
    B->>A: REST request, Authorization: Bearer ID token
    B->>A: WebSocket connect, ?token=ID token
```

## How it works

1. At startup the browser app fetches `appConfig.json` from its own origin and configures
   Amplify Auth with the pool, the stack's app client, the hosted domain and the redirect URL
   (`frontend/enc-notes/src/components/Authentication.js`). The sign-in button calls Amplify's
   redirect with Google as the provider, so the hosted domain passes the browser straight on to
   Google. Google returns to the pool, the pool returns the browser to the redirect URL with an
   authorization code, and Amplify exchanges it for tokens.
2. Every REST request carries the ID token as a bearer token. The REST API's Cognito authorizer
   checks it on every route, and each function takes the user id from its `sub` claim.
3. The WebSocket connect URL carries the ID token in its `token` parameter. The connect function
   reads the user id from it and refuses one without a user id.
4. Sign-out goes through Amplify, which clears the tokens and ends the session at the hosted
   domain, returning to the sign-out URL.

The stack's app client, `UserPoolClient` in `scripts/template.yaml`, is all this repo adds to the
pool: the authorization code flow, Google as its only identity provider, and these two URLs as
both its callback and its sign-out URLs:

- `https://<distribution>.cloudfront.net`, the frontend distribution's address and the stack's
  `CloudFrontUrl` output (the custom domain instead, when the `CustomDomainName` and
  `CertificateArn` parameters are both set);
- `http://localhost:3000`, for `npm start` in `frontend/enc-notes/`.

Both are without a trailing slash. `deploy.sh` writes the matching one into
`frontend/enc-notes/public/appConfig.json`: the CloudFront address while it builds the published
app, `http://localhost:3000` when it finishes, so a local `npm start` signs in against the
branch's stack. The CloudFront address is written even when a custom domain is set.

## Setting up Google

A fresh account, with no Google Cloud project and no OAuth client yet, starts in the
vpc-subnets-sgs repo, before any step of this one: the Setting up Google section of
vpc-subnets-sgs's docs/sign-in.md walks from no Google project at all to a deployed shared pool.
The Google side belongs to that pool. Its OAuth client redirects only to the pool's hosted
domain, `https://vsdb-<account id>.auth.<region>.amazoncognito.com/oauth2/idpresponse`, never
to this app, so this repo holds no Google client id or secret, and nothing of this app is
entered in the Google Cloud console.

1. **Check for the pool.** Run `./scripts/aws-helper.sh --list-resources`; a deployed shared
   pool shows as `vsdb-cognito-UserPool` with its id and its `vsdb-<account id>` domain. The pool
   must be in this app's region, the region of `scripts/lib/aws-config.sh`, since the authorizer
   names the pool in the stack's own account and region. With the pool there, go to step 3.
2. **Without the pool, set up Google and the pool first.** Follow the Setting up Google section
   of vpc-subnets-sgs's docs/sign-in.md from its first step. It creates the Google Cloud project,
   its consent screen and OAuth client, stores the client id and secret in the Secrets Manager
   secret `/vsdb-cognito/google-secrets`, and deploys the pool with `./scripts/deploy.sh
   --cognito` from that repo's `main`, so the stack is `vsdb-cognito` and the domain prefix
   `vsdb`. Then run step 1 again for the pool's id.
3. **Name the pool.** In `stack_parameter_overrides` of `scripts/lib/stack-params.sh`, set
   `ExistingUserPoolId` to the pool's id from step 1. `ExistingCognitoDomain` derives itself as
   `vsdb-<account id>.auth.<region>.amazoncognito.com` and needs no edit for the shared pool. A
   different pool works too when it has a hosted domain, which then goes into
   `ExistingCognitoDomain`, and a Google identity provider named exactly `Google`. Both
   parameters are required: the template creates no pool of its own when they are empty.
4. **Deploy.** `./scripts/deploy.sh --frontend` creates the app client with the URLs above and
   prints the CloudFront address; sign in there or at `http://localhost:3000`.

## Who may sign in

Google decides first: while the shared Google client's consent screen is in testing, only the
test users listed under **Audience → Test users** of that Google Cloud project get through, as
the vpc-subnets-sgs page describes. The shared pool runs no function before sign-up, so every
account Google lets through becomes a pool user on its first sign-in, and the app has no
allow-list of its own. Every signed-in user is an ordinary user who sees only their own notes,
since every query is keyed by the user id. A person who signs in to another app on the same pool
has the same user id here.
