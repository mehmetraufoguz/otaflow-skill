---
name: otaflow
description: Publish, promote, roll back and pause over-the-air (OTA) updates for an Expo app with Otaflow and its `otaflow` command line. Use when the user wants to ship a JavaScript update to installed Expo apps without a store release, set up Otaflow for a project (login, link an app, create the code-signing key, configure expo-updates), publish to preview or production, approve or request a promotion, undo a bad update, pause a channel, create CI keys, or debug an Otaflow error.
license: MIT
---

# Otaflow

Otaflow is an update server for Expo apps (the Expo Updates protocol) with a dashboard at https://otaflow.com and a command line, `otaflow`. An update is signed **on the developer's machine or in their CI** with a code-signing key that Otaflow never sees. Two channels exist: `preview` (every publish lands here first) and `production` (what users get).

More detail: `references/commands.md` (every command and flag), `references/concepts.md` (how it works, roles, rollbacks), `references/ci.md` (CI setup), `references/troubleshooting.md` (error messages and fixes).

## When helping someone

1. **Find out the situation first.** Run `otaflow whoami`, look for `.otaflow/project.json` (the folder is linked to an app) and `certs/certificate.pem`, and check `expo-updates` is in `package.json`. Don't assume.
2. **Never touch the private signing key.** Do not print it, read it into the conversation, copy it into the project, commit it, or put it in an environment file. The certificate (`certs/certificate.pem`) is public and is committed; the private key lives in the user's config folder (or `OTAFLOW_SIGNING_KEY` / `OTAFLOW_SIGNING_KEY_PATH` in CI secrets).
3. **Ask before anything that reaches users or changes access:** promoting to production, publishing straight to `production`, any `rollback`, `pause` / `resume`, `keys revoke`, `apikey create` / `delete`. Say what it will do and to which channel. Publishing to `preview` is safe to run.
4. **Verify with commands, don't guess.** `otaflow list` shows each channel, its head update and recent updates. Read what it prints.
5. **Prefer `--json`** when you need to parse output; every command supports it.
6. **Do not work around a refusal.** If the CLI refuses (dirty git tree, a value from `.env` in the bundle, a role that may not do something), explain why and fix the cause. Never pass `--new-runtime` just to get past an error; see `references/troubleshooting.md`.

## Install and sign in

Node.js 22 or newer.

```bash
npx otaflow --help           # or: npm install --global otaflow
otaflow login                # prints a code and a link: the user opens it, checks the code, clicks Allow
otaflow whoami
```

## Set up a project (once)

```bash
otaflow apps create --name "My App" --slug my-app    # the account owner creates apps; the slug is permanent
cd <the Expo project>
otaflow init --app my-app                            # writes .otaflow/project.json
otaflow keys init                                    # makes the key, writes certs/certificate.pem, registers the public half
```

`keys init` keeps the private key **outside** the project and prints where. Tell the user to back it up (a password manager): Otaflow cannot recover it, and without it no update reaches installed apps until a new binary ships.

Then `app.json` needs the update URL, the certificate and a runtime policy, and a **release build** must be made once with it (updates reach that binary from then on):

```json
{
  "expo": {
    "runtimeVersion": { "policy": "fingerprint" },
    "updates": {
      "url": "https://ota.otaflow.com/u/<slug>/manifest",
      "codeSigningCertificate": "./certs/certificate.pem",
      "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" }
    }
  }
}
```

The binary also has to ask for a channel. If the app is built with EAS, set `channel` per build profile in `eas.json` and add `"requestHeaders": { "expo-channel-name": "production" }` (or `preview`) under `updates` for builds that should follow that channel.

## Every day

```bash
git commit -am "Fix the checkout crash"                  # the tree must be clean
otaflow publish -m "Fix the checkout crash"              # to preview: exports, signs, uploads what changed
otaflow list                                             # check the head of preview
otaflow promote                                          # preview -> production
```

- `publish` refuses a dirty git tree, a commit that is already a channel's head, and a bundle that contains values from the project's `.env` files.
- Devices download an update at their **next launch** and run it on the one **after** that.
- `promote`: the account **owner** promotes directly. Anyone else (a member, or an API key) creates a request; the owner or the manager approves with `otaflow promotions approve --request <id>` or in the dashboard. `promote --wait` waits for the decision.

## When an update is wrong

| Situation | Do this |
|---|---|
| It must stop spreading now | `otaflow pause --channel production` (no key needed; installs keep what they run). `resume` when ready. |
| The latest update is broken | `otaflow rollback --channel production --previous` |
| Go back to the JavaScript inside the binary | `otaflow rollback --channel production --embedded` |
| Go back to a specific older update (owner) | `otaflow rollback --channel production --to <updateId>` |

A rollback is a **new** update carrying an older update's content, so devices move to it like any update. The **manager** can only undo the latest update. Each publish signs, ahead of time, the rollbacks that could undo it, so a manager without the key can run `otaflow rollback --previous` (or use the dashboard's Undo button); `--prepared` forces that path. Never roll back across an update that carried a database migration: publish a fix instead.

## CI

Use a key scoped to one app and some actions, never a person's login. See `references/ci.md`.

```bash
otaflow apikey create --app my-app --name "GitHub Actions" --actions publish,request_promote   # shown once
```

A key can publish to preview and request a promotion; it can never approve one.
