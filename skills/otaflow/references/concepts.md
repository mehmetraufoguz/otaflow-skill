# How Otaflow works

## The pieces

- **App**: one Expo app. Its update URL is `https://ota.otaflow.com/u/<slug>/manifest`. The slug is permanent.
- **Account**: owns apps and has a plan that sets seats and storage. An app can keep its updates in the customer's own bucket instead (`otaflow storage connect`, chosen before the first publish); those bytes do not count against the plan.
- **Channels**: `preview` and `production`. A binary asks for one channel (the `expo-channel-name` request header).
- **Runtime version**: a hash of the native layer (`runtimeVersion: { policy: "fingerprint" }`). An update only reaches binaries with the same runtime, so a JavaScript-only change reaches installed apps and a native change needs a new binary.
- **Code signing**: every manifest is signed with the app's private key; the binary carries the certificate and refuses anything not signed by it. Otaflow stores only the public key and checks each signature when you publish.

## An update's life

1. `otaflow publish`: the CLI exports the bundle, uploads only files the platform does not have (assets are named by SHA-256), signs the manifest on the machine, and the API verifies and stores it as the head of `preview`.
2. Test it on a preview build.
3. `otaflow promote`: the same signed bytes become the head of `production`. Nothing is exported or signed again, so production runs exactly what preview ran.
4. Devices check for an update at launch, download it in the background, and run it on the **next** launch.

## Who may do what

| | Owner | Manager | Member | API key |
|---|---|---|---|---|
| Publish to preview | yes | no | yes (apps they belong to) | with `publish` |
| Publish to production | yes | no | no | no |
| Promote | directly | approves requests | requests | with `request_promote` (request only) |
| Approve a promotion | yes | yes | no | never |
| Roll back | any earlier update, or embedded | the latest update only | no | with `rollback` |
| Pause / resume | yes | yes | no | with `pause` |
| Members list, API keys, plan | yes | read the members list and the activity log | no | no |

An account has one owner, at most one manager, and members. A member sees only the apps they were added to.

## Rollbacks

Devices launch the newest update they hold, so serving an old manifest again would change nothing. A rollback is therefore a **new** update carrying the old content (or a `rollBackToEmbedded` directive). Because it needs the signing key, each publish also signs, at that moment, the two rollbacks that could undo it, dated one second after it (a device only compares dates). Those are stored, and the owner, the manager or a key with `rollback` can activate one later without the key. They are valid only while that update is still the channel's head and the update it returns to is still the one before it.

## Safety properties

- The private key never reaches Otaflow. A compromised platform cannot make a device run code you did not sign.
- Devices verify the signature on every manifest and the hash of every asset; a tampered manifest or file is refused and the device keeps what it runs.
- Pausing a channel stops updates within seconds (the edge caches a decision for about 10 seconds) and undoes nothing.
- A revoked signing key cannot publish again, and **updates it signed are no longer handed to devices**: for each channel the platform serves the newest update signed by a key that is still trusted (or none). `otaflow keys revoke` lists the heads that stopped, and `otaflow list` marks them. Devices that already run an update signed by that key keep it, and binaries that trust only that key's certificate cannot verify another key's updates: moving on takes a new binary. Revoke a key only when it must not be trusted any more, not as a way to rotate while shipped binaries still depend on it.
