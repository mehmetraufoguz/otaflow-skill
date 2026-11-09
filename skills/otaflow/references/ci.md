# Otaflow in CI

Use an **API key scoped to one app**, and keep the signing key in the CI secret store. Never use a person's login in CI.

## 1. Create the key (the account owner, on their machine)

```bash
otaflow apikey create --app my-app --name "GitHub Actions" --actions publish,request_promote
```

The key is printed once. Store it as the secret `OTAFLOW_API_KEY`. Store the private signing key's PEM text as `OTAFLOW_SIGNING_KEY` (its location is printed by `otaflow keys init`; copy it from there into the secret store, nowhere else).

A key can publish to preview and ask for a promotion. It cannot publish straight to production and can never approve a promotion. Add `rollback` or `pause` to `--actions` only if the pipeline must do those.

## 2. A GitHub Actions example

```yaml
name: ota
on:
  push:
    branches: [main]
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx otaflow publish -m "$COMMIT_MESSAGE"
        env:
          OTAFLOW_API_KEY: ${{ secrets.OTAFLOW_API_KEY }}
          OTAFLOW_SIGNING_KEY: ${{ secrets.OTAFLOW_SIGNING_KEY }}
          # Public values the app reads at build time: publish reads no .env file.
          EXPO_PUBLIC_API_URL: ${{ vars.EXPO_PUBLIC_API_URL }}
          # Passed through the environment, never interpolated into the script (a commit message can contain anything).
          COMMIT_MESSAGE: ${{ github.event.head_commit.message }}
      - run: npx otaflow promote --wait
        env:
          OTAFLOW_API_KEY: ${{ secrets.OTAFLOW_API_KEY }}
```

`.otaflow/project.json` and `certs/certificate.pem` are committed; the key and the API key are not. The publish step must run in the project folder (set `working-directory` in a monorepo), and its git tree must be clean: install dependencies with `npm ci`, which does not modify tracked files. `promote --wait` creates a request and waits for the owner or the manager to approve it in the dashboard or the CLI; drop it if the pipeline should not block.

## 3. Rotating or removing

- Delete a key: `otaflow apikey delete --app my-app --id <keyId>` (find the id with `otaflow apikey list --app my-app`). Whatever uses it stops working at once.
- Replace a signing key: `otaflow keys init --key-id v2` makes and registers a new key under a **new** id (an id is never reused). Binaries already shipped keep trusting the old certificate, so keep the old key active until they are gone, and name `v2` in `codeSigningMetadata.keyid` (and the certificate path) of new builds. Revoking the old key (`keys revoke`) stops publishes signed with it.
