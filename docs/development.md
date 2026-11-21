# Development & deployment

This document details the local setup, the tests and the operations scripts behind the
[README](../README.md)'s Getting started and Scripts sections; [Architecture](architecture.md)
describes what the scripts deploy. Each script's header comment is its full documentation, and
`--help` prints it.

**Contents:** [Working copy location](#working-copy-location) · [Frontend](#frontend) ·
[Configuration](#configuration) · [Backend tests](#backend-tests) ·
[Deployment scripts (scripts/)](#deployment-scripts-scripts) ·
[What must exist outside the stack](#what-must-exist-outside-the-stack) · [Layers](#layers) ·
[Logs](#logs)

## Working copy location

Keep the checkout on the WSL ext4 filesystem (`~/projects/serverless/EncNotes`) rather than
under `/mnt/c`. WSL reaches Windows drives through a file-sharing protocol that makes every file
open a round trip to the Windows side, and `npm ci` and `sam build` open thousands of files.

`node_modules` folders record absolute paths, so each checkout installs its own instead of
receiving a copy.

## Frontend

```bash
./scripts/lib/app-config.sh     # on a fresh checkout: point appConfig.json at the branch's stack
cd frontend/enc-notes && npm ci && npm start
```

The dev server listens on `http://localhost:3000`, a callback the stack's Cognito app client
allows, and calls the deployed backend named in `frontend/enc-notes/public/appConfig.json`,
which `lib/app-config.sh` writes from the stack's outputs and every deploy rewrites.
`frontend/enc-notes/appConfig.example.json` shows the file's shape. A bare `npm run build`
deploys nothing; the production frontend ships only through `deploy.sh --frontend`.

`npm test` runs the Create React App test runner over the template's default test.

## Configuration

`./scripts/init-env.sh` creates the gitignored settings files under `scripts/lib/` from their
`.example.sh` files; each setting's comment says what it holds and whether it is required.

- `aws-config.sh` — the access key pair and region of the target account.
- `secrets.sh` — the Redis password the stack is deployed with.

Every other setting is committed in `scripts/lib/stack-params.sh`, which lists every parameter
the stack is deployed with: `sam deploy` keeps a stack's previous value for any parameter the
overrides leave out. The parameters it leaves out stay at their template defaults: the
networking, CORS and custom domain parameters. The app's name (`en`), its region and the log
retention are in `scripts/lib/app-env.sh`.

Which run a change needs: `scripts/template.yaml`, a settings file, a function or a layer takes
`./scripts/deploy.sh`; the React app takes `./scripts/deploy.sh --frontend-only`; both take
`./scripts/deploy.sh --frontend`, which deploys the backend first.

A bare `aws` command in your own shell has no credentials until it sources
`scripts/lib/aws-config.sh`; the scripts source it for you. A script that loads a settings file
first stops if git would track the file or a tracked file holds one of its secrets. It also
sets `core.hooksPath` to `.githooks`, whose `pre-commit` hook makes the same check on every
commit's staged changes.

## Backend tests

`scripts/test.sh` is the whole quality gate: `bash -n` on every script
and the pre-commit hook, then `sam validate --lint` on the template. It needs `sam` and no AWS
access. The Lambdas have no automated tests.

## Deployment scripts (`scripts/`)

Install the AWS CLI v2, the AWS SAM CLI, Node.js with npm, `jq` and git, each on `PATH`;
`deploy.sh` stops naming the first one missing.

No script takes an environment argument. The git branch selects the stack every script acts on:
`main` targets `en` (stage `prod` of both APIs), any other branch `en-fb` (stage `dev`); an
exported `STACK_NAME` wins. The two stacks differ in the origin they allow
([The allowed origin depends on the stack](design-patterns.md#the-allowed-origin-depends-on-the-stack)),
and work on the same notes
([Settings come from the template and the stack's outputs](design-patterns.md#settings-come-from-the-template-and-the-stacks-outputs)).

- `deploy.sh` — validates and lints `scripts/template.yaml`, runs
  `sam build --cached --parallel`, which rebuilds only the functions whose source changed, then
  `sam deploy` with the parameters of `stack-params.sh`. `--frontend` also builds the React
  app, syncs it to the frontend bucket and invalidates the CloudFront distribution;
  `--frontend-only` does only that; `--sync` runs `sam sync` and is refused on `main`, since it
  deploys without a change set to review. Every run rewrites `appConfig.json` through
  `lib/app-config.sh`, stamps its `BUILD` value with the stack
  name and the time, and ends with the Cognito redirect at `http://localhost:3000`.
- `aws-helper.sh` — AWS operations on the branch's stack, the same
  file in every AWS repo: function logs, log retention, `--delete-stack [--empty]`,
  `--list-resources`, `--xray on|off` and `--prune-layers`. This repo's own option is in
  `lib/aws-helper-app.sh`: `--apigateway-logging-role`
  deploys `scripts/apigateway-logging-role.yaml` as the stack `api-gateway-logging-role` and
  sets its role as the account's API Gateway CloudWatch role, once per account and region.
- `test.sh` — the quality gate, above.
- `tools/` — hands-on tools: `publish-docs.sh` (publishes the
  README and `docs/` to the `EncNotes-public` repository).

`scripts/lib/app-env.sh` holds what the scripts share: the `--help` printer, the settings-file
loaders, which refuse a file git tracks, and the branch's stack selection. `sam` writes its
build folder `.aws-sam/` into `scripts/`; it is gitignored.

## What must exist outside the stack

A deploy depends on parts no script of this repo creates, which `stack-params.sh` names:

- The shared stack `vsdb-cognito`: the user pool with Google sign-in and its domain, an
  authorized redirect of the Google OAuth client. The stack adds its own app client to that
  pool; [Sign-in](sign-in.md#setting-up-google) covers the pool in a new account.
- Redis on the account's shared database host, run by the vpc-subnets-sgs repository, reached
  by the host's public name with the password of `secrets.sh`.
- The notes and users tables and the notes KMS key, which belong to no stack
  ([Settings come from the template and the stack's outputs](design-patterns.md#settings-come-from-the-template-and-the-stacks-outputs)).
- The account's API Gateway CloudWatch role, set once per account and region with
  `./scripts/aws-helper.sh --apigateway-logging-role`, so the WebSocket stage may write logs.

## Layers

The encryption layer at `backend/notes/layers/encryption/` is a git submodule. Without
`git submodule update --init --recursive` after a plain clone, the SAM build has no layer to
package; a change to it is made in its own repository.

The AWS SDK v3, ioredis and jsonwebtoken layers deploy from the zips committed in
`backend/layers/`; no script rebuilds them, so a change to a `nodejs/package.json` there
reaches Lambda only after its zip is rebuilt by hand. Which layers there are and how a function
loads one is in [Shared code lives in layers](design-patterns.md#shared-code-lives-in-layers).

## Logs

Each function writes to a log group the template declares, named and kept as
[The entry points](architecture.md#the-entry-points) states, so a deploy leaves the logs alone;
`aws-helper.sh --delete-stack` deletes them with the stack.
`./scripts/aws-helper.sh --logs <function> --since 1d --errors` shows one function's warnings
and errors; `--trim-logs` deletes the stack's log streams on request.
