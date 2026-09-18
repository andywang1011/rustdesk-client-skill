# build-windows-rustdesk-client

Replica of the ModelScope agent skill [build-windows-rustdesk-client](https://www.modelscope.cn/skills/zhm1321391634/build-windows-rustdesk-client) by **zhm1321391634** (Apache License 2.0), mirrored here so it can be used directly with agent tools (Codeshift/Codex/Gemini CLI/Claude Code/opencode, …) via the [Agent Skills](https://agentskills.io) convention.

## What it does

Builds a custom **RustDesk Windows client** with user-specified **ID server, relay server, API server, public key and fixed password**, then pushes it to **GitHub Actions** for compilation:

```
User gives config → create workflow YAML → create git repo → push → trigger Actions → return URL
```

Sensitive values (server addresses, password, public key, …) never appear in the public repository — they are stored as encrypted **GitHub Secrets** and only referenced in the workflow via `${{ secrets.XXX }}` placeholders. The public-key / password / server config is compiled into the binary.

## Contents

```
.agents/skills/build-rustdesk-client/
├── SKILL.md                     # step-by-step instructions for the agent
└── references/
    └── workflow-template.yml    # GitHub Actions build workflow (two jobs)
```

## Usage

1. Point your agent at `.agents/skills/build-rustdesk-client/` (auto-discovered by most agent tools).
2. Have it follow `SKILL.md`. Prerequisites: `git config` set and `gh auth login` done.
3. Provide the 5 config values:
   - `ID_SERVER` (required)
   - `RELAY_SERVER` (required)
   - `API_SERVER` (optional, e.g. `http://your-server.com:21114`)
   - `PUBLIC_KEY` (optional; empty disables encryption)
   - `FIXED_PASSWORD` (optional; unattended-access password)

## Notes

- The workflow builds against upstream `rustdesk/rustdesk` (master) and requires a **public** repo for free GitHub Actions.
- `VCPKG_COMMIT_ID` is pinned to `9e593bb18ea69cc5095e012465dcd675a822ed0d` to match the current `vcpkg.json` baseline in `rustdesk/rustdesk`. If upstream bumps the baseline, update both in lockstep (see `flutter-build.yml` upstream).
- If the upstream `libs/hbb_common/src/config.rs` patterns change, update the Python regexes in `references/workflow-template.yml` (see the "Upstream Verification" section of `SKILL.md`).

## Client customization reference

- [`客户端修改参考.md`](./客户端修改参考.md) — a consolidated, formatted reference of all
  source-level client customizations (server/account, Flutter UI trimming, home-page
  logo, default options, security/password).

## Attribution / License

Original skill: <https://www.modelscope.cn/skills/zhm1321391634/build-windows-rustdesk-client>

This repository is a mirror under the **Apache License 2.0** (see [LICENSE](./LICENSE)).