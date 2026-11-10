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
| `This account has no active plan: publishing is read-only.` | The account has no active plan (or it lapsed). Publishing and uploads are closed until a plan is active; devices keep getting what is already published. Ask the account owner or Otaflow. |
| `That update's files were cleaned up to free space...` | A rollback or promotion of an update whose files were deleted by `otaflow storage gc`. Publish it again from its commit. |
| `Platform storage is full (... bytes used; this needs ... more). Free space, add a storage pack, or connect your own bucket.` | The account's storage is full. Nothing stored is lost and devices still get updates, but no new bytes are accepted. |
| `The platform must be ios or android.` | Another value was passed to `--platform`. |
| `This commit is already the head of <channel>: commit the change you meant to ship.` | Nothing new since the last publish of this runtime; the CLI says so before exporting. |
| `That key id is already used on this app (revoked ids stay taken): choose another.` | A key id is used once. Pick a new `--key-id` for the new key. |
| `The signature does not verify under key ...` | The manifest was signed with a different key than the one named. Use the matching `--key-id` and key. |

## Promotion and rollback

| Message | Meaning and fix |
|---|---|
| `You may not do that on this app.` | The role or key lacks the action (see the table in `concepts.md`). |
| `That update is already on production.` | It is the update production runs now. Publish a new update to ship more. |
| `That update already went to production and has been replaced or rolled back since...` | A promotion is not repeated. Commit a change and publish again. |
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

Work down this list when a build does not pick up an update (`references/setup.md` has the details):

1. **Is it a release build** (not Expo Go, not a debug or development-client build)?
2. **Does the binary ask for the channel you published to?** `expo-channel-name` in `updates.requestHeaders`; with none, the server answers "no update".
3. **Is the runtime version the same?** Compare `npx expo-updates runtimeversion:resolve --platform ios` for the build's configuration with the runtime in `otaflow list`. A different one (a native change, or a fingerprint split between profiles) means the update is for another binary.
4. **Does the binary hold the certificate for the key that signed?** `codeSigningCertificate` and `codeSigningMetadata.keyid` at build time must match the key used to publish; otherwise the device refuses every update.
5. **Is the channel paused?** `otaflow list` shows PAUSED; `otaflow resume`.
6. **Did you open the app twice?** The first launch downloads, the second runs it.
7. **Is the head what you expect?** `otaflow list`; a rollback or a newer promotion may have replaced it.

Devices download at launch and run an update at the **next** launch: close the app fully and open it twice. A device that refuses an update keeps what it runs: that happens for a manifest not signed by the key in its certificate, a manifest or file that was changed, or a different runtime version. A runtime version change (anything native) needs a new binary.

## Your own bucket

| Message | Meaning and fix |
|---|---|
| `The bucket refused to store an object: check the access key and its permissions.` | The key cannot write there. It needs read, write and delete on the bucket (or the `--prefix`). |
| `The key can write but not delete: grant delete access.` | The test object could not be removed; add delete permission. |
| `The bucket or path was not found ...` | Wrong bucket name, endpoint or region. R2 uses region `auto`; S3 refuses a wrong region. |
| `This app already stores updates. Choose the storage before the first publish: existing assets are not moved.` | Storage is chosen before an app's first publish. Create a new app to use a bucket from the start. |
| `This app already stores updates in that bucket, and they are not moved: it cannot go back to platform storage.` | Disconnect only works while nothing is stored. |
| `The endpoint must use https.` / `must be a public hostname.` | Only https endpoints on public hostnames are accepted. |
| `The storage encryption key is not configured.` | The Otaflow service itself is missing its sealing secret: a platform problem, not the customer's. |
| A device says the update failed to download after the bucket changed | `otaflow storage check`; a revoked or rotated bucket key makes downloads fail while devices keep running what they have. |

## A channel serves an older update than the head

`otaflow list` shows `signed by a revoked key: not handed to devices; they get <id> instead` under a head. The key that signed it was revoked, so devices are handed the newest update signed by a trusted key (or none). Publish a new update with an active key to replace it; devices whose binary trusts only the revoked key's certificate need a new binary.
