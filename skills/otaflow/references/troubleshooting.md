# Otaflow troubleshooting

Messages are shown as the CLI prints them. Fix the cause; do not work around a refusal.

## Publishing

| Message | Meaning and fix |
|---|---|
| `Commit these first (an update that matches no commit cannot be reproduced...)` | The git tree is dirty. Commit the listed files, then publish again. |
| `This commit is already the channel's head: commit the change you meant to ship.` | Nothing new since the last publish. Make and commit a change. |
| `A value from your .env files reached the bundle...` | A value from a `.env*` file (not an example/template file) ended up in the JavaScript. Remove it from the code path that reads it, or set it through the app config. Do not publish around it. |
| `No device has asked for runtime <hash> yet, so this update would reach nobody.` | No installed binary with this runtime has contacted Otaflow. Open a release build that has this runtime once, then publish. Use `--new-runtime` only if the build is on its way and the user says so. |
| `No active signing key "main" on this app` | The key was revoked or never registered. `otaflow keys list`; `otaflow keys init` registers one. |
| `Could not read the private key at ...` | The key is not where the CLI looks. Pass `--key`, or set `OTAFLOW_SIGNING_KEY` / `OTAFLOW_SIGNING_KEY_PATH`. |
| `expo-updates` errors from `runtimeversion:resolve` | `expo-updates` is not installed in the project, or the project is not an Expo app. `npx expo install expo-updates`. |
| `Only the account owner publishes straight to production; promote from preview.` | Publish to preview, then `otaflow promote`. |
| The account plan refuses uploads | The account has no active plan (or it lapsed). Publishing is closed until the plan is active; devices keep working. Contact the account owner or Otaflow. |
| A storage error (HTTP 507) | The account's platform storage is full. Nothing already stored is lost and devices still get updates, but no new bytes are accepted. |
| `The signature does not verify under key ...` | The manifest was signed with a different key than the one named. Use the matching `--key-id` and key. |

## Promotion and rollback

| Message | Meaning and fix |
|---|---|
| `You may not do that on this app.` | The role or key lacks the action (see the table in `concepts.md`). |
| `That update is already on production.` | It was promoted (or rolled back across) already. Publish a new update to ship more. |
| `Production already runs something newer than this update.` | Promotions only move forward. Publish again. |
| `This request expired after 48 hours: ask again.` | Run `otaflow promote` again. |
| `A manager can undo only the latest update: go back to the one before it, or to the embedded JavaScript.` | Managers may use `--previous` or `--embedded` only. |
| `No rollback was signed for this update (it came before prepared rollbacks, or from a rollback itself): use the CLI with the signing key.` | The head has no rollback signed at publish (it is itself a rollback, or an older update). Roll back with the private key. |
| `The update this rollback returns to is no longer the one before the head...` | Something moved since the update was published. Roll back with the private key. |

## Login and access

| Message | Meaning and fix |
|---|---|
| `Not signed in. Run otaflow login.` | Run `otaflow login`, or set `OTAFLOW_API_KEY` in CI. |
| `This API key is not valid.` | The key was deleted or expired. Create another. |
| The login code page says the code is not valid | Codes expire after a few minutes. Run `otaflow login` again. |
| `Could not read the OS keychain.` | Use `OTAFLOW_CREDENTIALS=file` to keep the login in a file. |

## On the device

Devices download at launch and run an update at the **next** launch: close the app fully and open it twice. A device that refuses an update keeps what it runs: that happens for a manifest not signed by the key in its certificate, a manifest or file that was changed, or a different runtime version. A runtime version change (anything native) needs a new binary.
