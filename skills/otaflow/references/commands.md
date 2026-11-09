# otaflow commands

Every command takes `--json` (machine-readable output) and `--api-url <url>` (or `OTAFLOW_API_URL`; default `https://api.otaflow.com`). `otaflow <command> --help` (for example `otaflow keys init --help`) prints a command's flags for the installed version; they are listed here too.

## Account and apps

| Command | What it does |
|---|---|
| `otaflow login [--no-browser]` | Sign in through the browser: shows a code and a link (and opens it, unless `--no-browser` or on CI); the user confirms the code and clicks Allow. |
| `otaflow whoami` | The signed-in user. |
| `otaflow logout` | Forget this machine's login and end the session. |
| `otaflow apps list` | The apps the user can see, with ids and the user's role. |
| `otaflow apps create --name <n> --slug <s> [--bundle-id <id>]` | Create an app in the user's own account (owner only). The slug is part of the update URL and is never reused. |
| `otaflow init --app <slug-or-id>` | Link the current folder to an app (writes `.otaflow/project.json`). |

## Signing keys

The private key never leaves the user's machine.

| Command | What it does |
|---|---|
| `otaflow keys init [--key-id main] [--key-dir <dir>] [--cert-dir <dir>] [--years <1-30>] [--force]` | Generate a 2048-bit RSA key and certificate, register the public half, write the certificate to `./certs` (`certificate.pem` for `main`, `<id>.certificate.pem` otherwise) and the private key to the user's key folder (`~/.config/otaflow/keys/<appId>/<id>.key.pem`). The key is written to disk **before** registering, so a network failure never loses it. It refuses if that private key file already exists. `--force` overwrites the local file with a new key, which is only useful when that id was never registered (a failed first run): the API refuses to register an id twice, so a rotation always uses a new `--key-id`. |
| `otaflow keys register --key <private.pem> --cert <certificate.pem> [--key-id <id>]` | Register an existing key and certificate (for example one that already shipped in binaries). Both files are required: the key signs a proof of possession; the private key is not uploaded. |
| `otaflow keys list` | Keys registered for the app and whether each is active or revoked. |
| `otaflow keys public [--key <private.pem>]` | Print the fingerprint of a local private key. |
| `otaflow keys path [--key-id main]` | Where the private key is on this machine, for the backup. Prints the path, never the key. |
| `otaflow keys revoke --key-id <id>` | Stop accepting publishes signed with the key (owner). Installed apps that trust its certificate keep running what they run. |

A key id is used once per app: a revoked id stays taken, so a new key always gets a new `--key-id` (and new builds name it in `codeSigningMetadata.keyid`). The key is found in this order: `--key <path>`, `OTAFLOW_SIGNING_KEY` (the PEM text), `OTAFLOW_SIGNING_KEY_PATH`, then the app's key folder in the user's config directory.

## Publishing

| Command | What it does |
|---|---|
| `otaflow publish [-m <message>] [--channel preview\|production] [--platform ios\|android] [--key-id main] [--key <path>] [--new-runtime]` | Export with `expo export`, hash and upload only the files the platform lacks, sign the manifest locally, publish. Default channel: `preview`. Straight to `production` is for the account owner only. |
| `otaflow list [--channel <c>] [--limit <n>]` | Each channel's head, paused state, runtimes seen and recent updates. |
| `otaflow pause --channel <c>` / `otaflow resume --channel <c>` | The brake: devices get no update while paused. Needs no key. |

What `publish` does, in order: checks the platform is `ios` or `android` (`android` is served but not yet tested on a device); needs a login or `OTAFLOW_API_KEY`; checks the git tree is clean **for the project folder** (`git status` limited to it); loads the signing key; resolves the runtime version with `npx expo-updates runtimeversion:resolve` (`expo-updates` must be installed); exports the bundle with `expo export` and **no `.env` files read**; scans the bundle for values from `.env*` files; asks the API which files it lacks and whether they fit the account's storage; uploads them (four at a time); signs the manifest locally; publishes. The API can still refuse: no active plan, the key unknown or revoked, a signature that does not verify, a stale manifest, the same commit as the head, or an unseen runtime. `--new-runtime` skips only the "no device has asked for this runtime" check.

## Promotion

| Command | What it does |
|---|---|
| `otaflow promote [--update <id>] [--platform ios\|android] [--wait] [--timeout <s>]` | Move the preview head (or `--update`) to production. The owner does it directly; others create a request (valid 48 hours). |
| `otaflow promotions list [--status pending\|approved\|rejected\|cancelled\|expired]` | Requests for the app. |
| `otaflow promotions approve --request <id>` | Owner or manager. The requester cannot approve their own. |
| `otaflow promotions reject --request <id> [--note <text>]` | Owner or manager. |
| `otaflow promotions cancel --request <id>` | The requester or the owner. |

## Rollback

`otaflow rollback --channel <c> (--previous | --to <updateId> | --embedded) [--runtime <v>] [--platform ios\|android] [-m <message>] [--key-id main] [--key <path>] [--prepared]`

- `--previous`: the update before the head. `--embedded`: devices return to the JavaScript inside their binary. `--to`: a specific earlier update (owner or a key with the `rollback` action).
- With the private key present, the rollback is signed now. Without it (or with `--prepared`), the rollback that was signed when the current head was published is activated instead; it is valid only while that update is still the head.
- A manager can undo only the latest update (`--previous` or `--embedded`).

## Your own bucket

By default Otaflow stores an app's updates and they count against the account's storage. An app can keep them in the customer's own S3-compatible bucket (S3, Cloudflare R2, Backblaze B2, ...) instead. Storage is chosen **before the first publish**: stored updates are not moved, so a bucket can only be connected or removed while the app has stored nothing. Connecting is for the account owner.

| Command | What it does |
|---|---|
| `otaflow storage status` | Where this app's updates are stored, and the last check of the bucket. |
| `otaflow storage connect --provider <r2\|s3\|b2\|gcs\|other> --endpoint <https url> --bucket <name> [--region <r>] [--prefix <folder>] [--public-url <https url>] [--access-key-id <id>] [--secret-stdin]` | Check the key (write, read and delete a test object) and save the bucket. The key id is `--access-key-id` or `OTAFLOW_BUCKET_ACCESS_KEY_ID`; the secret is on stdin with `--secret-stdin` or in `OTAFLOW_BUCKET_SECRET`: never a flag. |
| `otaflow storage check` | Run the test object again; exits 1 if it fails. |
| `otaflow storage disconnect` | Back to platform storage (only while nothing is stored). |

Rules for the agent: never put the secret key in a file, a flag or the conversation; have the user supply it through the environment or a secret manager. R2 uses region `auto`; S3 refuses a wrong region. Give the key only the bucket (or one `--prefix`) with read, write and delete. Devices download from a signed link, or from `--public-url` when the bucket is public or behind a CDN. The endpoint must be https and a public hostname.

## People and activity

| Command | What it does |
|---|---|
| `otaflow members list [--app <slug>]` | The account's members, roles and the apps each can reach. |
| `otaflow members invite` | A one-time invite link (valid a week). The person joins with it, then needs app access. |
| `otaflow members role --member <email\|id> --role <manager\|member>` | Owner only; one manager at a time. |
| `otaflow members access --member <email\|id> --to <app slug> [--revoke]` | Give a member an app, or take it away. |
| `otaflow members remove --member <email\|id>` | Remove them; the CI keys they made stop working. |
| `otaflow log [--limit n] [--action <prefix>] [--actor <text>] [--only-app <name>]` | The account's activity log, newest first (owner and manager). |

These act on the account that owns the linked project (or `--app`). They need a person's login, not a CI key.

## Checking a project

`otaflow doctor [--key-id main]` checks, in order: Node.js 22+, the project link, the login and app access, `app.json` (update URL for this app, certificate and key id, channel header, runtime policy), the private key on this machine, the certificate (valid, and the registered key's), `expo-updates`, the git tree and the runtime version (and whether any device has asked for it). It prints `ok` / `warn` / `FAIL` per line and exits 1 if anything failed. Run it first when something does not work.

## CI keys

| Command | What it does |
|---|---|
| `otaflow apikey create --app <slug> --name <n> [--actions publish,request_promote] [--expires-days <d>]` | Print a key (once, on stdout; notes go to stderr). Actions: `publish`, `request_promote`, `rollback`, `pause`. |
| `otaflow apikey list --app <slug>` | Keys of the app's account (never the secret). |
| `otaflow apikey delete --app <slug> --id <keyId>` | Delete a key: whatever uses it stops working at once. |

Use a key with `OTAFLOW_API_KEY=otk_...` in the environment; it replaces the login.

## Environment variables

| Variable | Meaning |
|---|---|
| `OTAFLOW_API_URL` | The Otaflow API address (default `https://api.otaflow.com`). |
| `OTAFLOW_API_KEY` | Use this API key instead of a login. |
| `OTAFLOW_SIGNING_KEY` | The private key PEM text (CI secret). |
| `OTAFLOW_SIGNING_KEY_PATH` | Path to the private key file. |
| `OTAFLOW_CREDENTIALS=file` | Keep the login in `~/.config/otaflow/credentials.json` (mode 0600) instead of the OS keychain. |
