# otaflow-skill

An agent skill for [Otaflow](https://otaflow.com): over-the-air updates for Expo apps, with a code-signing key that stays on your machine. It teaches an AI coding agent to set a project up, publish and promote updates, undo a bad one, pause a channel and wire CI with the `otaflow` command line, and to do it safely (never touching the private key, asking before anything that reaches users).

## Install

```bash
npx skills add mehmetraufoguz/otaflow-skill
```

or copy `skills/otaflow/` into your agent's skills folder.

## What is inside

| File | What it is |
|---|---|
| `skills/otaflow/SKILL.md` | The skill: when to use it, the rules for the agent, setup, the daily flow, rollbacks, CI. |
| `skills/otaflow/references/setup.md` | Project and build setup: `app.json`, channels, runtime versions, build-time values. |
| `skills/otaflow/references/commands.md` | Every command and flag. |
| `skills/otaflow/references/concepts.md` | How it works: channels, runtime versions, signing, roles, rollbacks. |
| `skills/otaflow/references/ci.md` | CI keys and a GitHub Actions example. |
| `skills/otaflow/references/troubleshooting.md` | Error messages and what to do. |

The command line is on npm (`npx otaflow --help`, MIT). This repository holds only the skill; the Otaflow service itself is not open source.

## License

MIT. See [LICENSE](LICENSE).
