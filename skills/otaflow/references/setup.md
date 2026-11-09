# Setting a project up so updates arrive

Most "my update does not reach devices" problems are decided here, before any publish. Check these in order.

## 1. The binary is a release build with the right `updates` block

`expo-updates` must be installed (`npx expo install expo-updates`) and `app.json` must carry:

```json
{
  "expo": {
    "runtimeVersion": { "policy": "fingerprint" },
    "updates": {
      "url": "https://ota.otaflow.com/u/<slug>/manifest",
      "codeSigningCertificate": "./certs/certificate.pem",
      "codeSigningMetadata": { "keyid": "main", "alg": "rsa-v1_5-sha256" },
      "requestHeaders": { "expo-channel-name": "production" }
    }
  }
}
```

- `url`: the app's slug on `ota.otaflow.com`. A binary compiled with an address keeps it forever: choose the slug carefully before shipping.
- `codeSigningCertificate`: the file written by `otaflow keys init` (commit it).
- `codeSigningMetadata.keyid`: must equal the key id that signs (`otaflow publish --key-id`, default `main`). After a key rotation, builds name the new id.
- `requestHeaders["expo-channel-name"]`: the channel this binary follows, `preview` or `production`. Without it the server answers "no update".
- A debug or development-client build, or Expo Go, never applies updates; test with a **Release** build (for example `npx expo run:ios --configuration Release`, or an EAS build).

## 2. One channel per build profile

EAS Build writes a profile's `channel` into the binary only when `updates.url` is on Expo's own host, so with Otaflow the header has to come from the app config. Either set it statically in `app.json`, or per EAS profile in `app.config.js`:

```js
const { readFileSync } = require("node:fs")
const { join } = require("node:path")

// The channel the active EAS build profile declares in eas.json (read as text, not required, so
// eas.json does not become a fingerprint input).
const channelFor = (profile) => {
  if (!profile) return undefined
  const eas = JSON.parse(readFileSync(join(__dirname, "eas.json"), "utf8"))
  const seen = new Set()
  let name = profile
  while (name && !seen.has(name)) {
    seen.add(name)
    const entry = eas.build?.[name]
    if (!entry) return undefined
    if (typeof entry.channel === "string") return entry.channel
    name = entry.extends
  }
}

module.exports = ({ config }) => {
  const channel = channelFor(process.env.EAS_BUILD_PROFILE)
  if (!channel) return config
  return {
    ...config,
    updates: {
      ...config.updates,
      requestHeaders: { ...config.updates?.requestHeaders, "expo-channel-name": channel }
    }
  }
}
```

A plain `expo run:ios` has no profile and gets no header, which the server answers with "no update": right for a build no channel was meant for.

## 3. Preview and production builds must share one runtime version

A promotion moves the **same signed update** from `preview` to `production`; it only reaches a production binary whose runtime version equals the one it was built for. With `runtimeVersion.policy: "fingerprint"` the runtime is a hash of the native layer and the Expo config, so some things split your builds onto different runtimes by accident. Check, for each build profile:

```bash
npx expo-updates runtimeversion:resolve --platform ios
```

The values must match between preview and production builds (and must not change from one build number to the next). Known causes of a split, measured with `@expo/fingerprint` 0.20.x (verify on your version):

- `version` and the iOS `buildNumber` are hashed, so an auto-incremented build number makes every build its own runtime.
- `package.json` scripts are hashed: adding an npm script changes the runtime.
- `eas.json` and `.gitignore` are hashed.
- **`updates.requestHeaders` is hashed**, and it differs between preview and production builds, so the two never match.
- The code-signing certificate is **not** hashed: a rotated key under the same path keeps the same runtime, though old binaries cannot verify the new key.

A `fingerprint.config.js` next to `app.json` fixes these:

```js
/** @type {import("expo/fingerprint").Config} */
module.exports = {
  // Skip version numbers and npm scripts and .gitignore as runtime inputs.
  sourceSkips: ["ExpoConfigVersions", "PackageJsonScriptsAll", "GitIgnore"],
  ignorePaths: ["eas.json"],
  // Hash the certificate file, so a new key gets a new runtime.
  extraSources: [{ type: "file", filePath: "certs/certificate.pem", reasons: ["codeSigningCertificate"] }],
  // Drop the channel header from the hashed config: it is read natively and changes no compiled code.
  fileHookTransform: (source, chunk) => {
    if (source.type !== "contents" || source.id !== "expoConfig" || typeof chunk !== "string") return chunk
    const expoConfig = JSON.parse(chunk)
    if (expoConfig.updates !== undefined) delete expoConfig.updates.requestHeaders
    return JSON.stringify(expoConfig)
  }
}
```

Native files the fingerprint cannot see (for example a widget's Swift sources copied during prebuild) must be added with `extraSources`, or a change to them will not move the runtime and an update could reach a binary that cannot run it.

## 4. Values the app needs at build time

`otaflow publish` runs `expo export` with `EXPO_NO_DOTENV=1`: **no `.env` file is read**, on purpose, so a developer's local address or key cannot ride along. The consequence: public values the JavaScript reads (`process.env.EXPO_PUBLIC_*`) must be supplied through the environment of the publish command, the same values a store build of that channel uses:

```bash
EXPO_PUBLIC_API_URL=https://api.example.com otaflow publish -m "..."
```

In CI, set them as step environment variables. If a needed value is missing, the update ships with it undefined.

The CLI also scans the bundle for the values in the project's `.env*` files (not `.env.example`-style templates) and refuses to publish if one is in it.

## 5. The channel and the runtime must have been seen

`publish` refuses an update for a runtime no device has asked about yet ("No device has asked for runtime ..."): that usually means the build does not match what was configured. Open a release build of that runtime once so it contacts Otaflow; then publish. `--new-runtime` skips this check and is only for a build that is on its way (an update for a runtime nobody has reaches nobody).

## 6. Checking it end to end

1. `otaflow list`: the channel shows the head you published and the runtime.
2. Close the release build completely and open it (the update downloads in the background); close and open it again (now it runs it). A visible change in the app, or `Updates.updateId` from `expo-updates`, tells you it ran.
3. Nothing arrived? `references/troubleshooting.md`, section "On the device".
