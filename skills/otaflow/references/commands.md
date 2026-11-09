# otaflow commands

Every command takes `--json` (machine-readable output) and `--api-url <url>` (or `OTAFLOW_API_URL`; default `https://api.otaflow.com`). Run `otaflow <command> --help` for the flags of your installed version.

## Account and apps

| Command | What it does |
|---|---|
| `otaflow login` | Sign in through the browser: shows a code and a link; the user confirms the code and clicks Allow. |
| `otaflow whoami` | The signed-in user. |
| `otaflow logout` | Forget this machine's login and end the session. |
| `otaflow apps list` | The apps the user can see, with ids and the user's role. |
| `otaflow apps create --name <n> --slug <s> [--bundle-id <id>]` | Create an app in the user's own account (owner only). The slug is part of the update URL and is never reused. |
| `otaflow init --app <slug-or-id>` | Link the current folder to an app (writes `.otaflow/project.json`). |

## Signing keys

The private key never leaves the user's machine.

| Command | What it does |
|---|---|
| `otaflow keys init [--key-id main] [--cert-dir ./certs] [--force]` | Generate a 2048-bit RSA key and certificate, register the public half, write the certificate to the project. |
| `otaflow keys register --cert <pem> [--key <pem>]` | Register an existing key (for example one that already shipped in binaries). |
| `otaflow keys list` | Keys registered for the app and whether each is active or revoked. |
| `otaflow keys public` | Print the fingerprint of a local private key. |
| `otaflow keys revoke --key-id <id>` | Stop accepting publishes signed with the key (owner). Installed apps that trust its certificate keep running what they run. |

The key is found in this order: `--key <path>`, `OTAFLOW_SIGNING_KEY` (the PEM text), `OTAFLOW_SIGNING_KEY_PATH`, then the app's key folder in the user's config directory.

## Publishing

| Command | What it does |
|---|---|
| `otaflow publish [-m <message>] [--channel preview\|production] [--platform ios] [--key-id main] [--key <path>] [--new-runtime]` | Export with `expo export`, hash and upload only the files the platform lacks, sign the manifest locally, publish. Default channel: `preview`. Straight to `production` is for the account owner only. |
| `otaflow list` | Each channel's head, paused state, runtimes seen and recent updates. |
| `otaflow pause --channel <c>` / `otaflow resume --channel <c>` | The brake: devices get no update while paused. Needs no key. |

`publish` checks, in this order: a clean git tree; the runtime version (`expo-updates` must be installed); a bundle free of `.env` values; the account's plan; the commit is not already the head. `--new-runtime` skips only the "no device has asked for this runtime" check.

## Promotion

| Command | What it does |
|---|---|
| `otaflow promote [--update <id>] [--platform ios] [--wait] [--timeout <s>]` | Move the preview head (or `--update`) to production. The owner does it directly; others create a request (valid 48 hours). |
| `otaflow promotions list [--status pending\|approved\|rejected\|cancelled\|expired]` | Requests for the app. |
| `otaflow promotions approve --request <id>` | Owner or manager. The requester cannot approve their own. |
| `otaflow promotions reject --request <id> [--note <text>]` | Owner or manager. |
| `otaflow promotions cancel --request <id>` | The requester or the owner. |

## Rollback

`otaflow rollback --channel <c> (--previous | --to <updateId> | --embedded) [--runtime <v>] [--platform ios] [-m <message>] [--key-id main] [--key <path>] [--prepared]`

- `--previous`: the update before the head. `--embedded`: devices return to the JavaScript inside their binary. `--to`: a specific earlier update (owner or a key with the `rollback` action).
- With the private key present, the rollback is signed now. Without it (or with `--prepared`), the rollback that was signed when the current head was published is activated instead; it is valid only while that update is still the head.
- A manager can undo only the latest update (`--previous` or `--embedded`).

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
