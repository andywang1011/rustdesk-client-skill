---
name: build-rustdesk-client
description: >
  Build a custom RustDesk Windows client with user-specified ID server, relay server, API server,
  public key, and fixed password, then push to GitHub Actions for compilation.
  Trigger this skill whenever the user asks to compile/build/customize a RustDesk client,
  mentions wanting a "fixed password" or "custom server" RustDesk build, or wants to
  create their own branded RustDesk release. This skill handles the entire process:
  collecting config, creating a GitHub repo, pushing the workflow, and triggering the build.
  Do NOT use this for general RustDesk questions or troubleshooting — only for BUILDING a client.
---

# Build Custom RustDesk Client

This skill automates building a custom Windows RustDesk client via GitHub Actions. It captures everything learned from a verified end-to-end build process.

## Workflow Overview

```
User gives config → You create workflow YAML → Create git repo → Push → Trigger Actions → Return URL
```

## Step 1: Collect Configuration from User

Ask the user for these values. **All are required unless noted.** If they don't provide one, ask clearly.

| Variable | Example | Notes |
|---|---|---|---|
| ID_SERVER | `your-server.com` | The rendezvous/ID server hostname (no protocol) |
| RELAY_SERVER | `your-server.com` | The relay server hostname (usually same as ID server) |
| API_SERVER | `http://your-server.com:21114` | Full URL with protocol and port. **Optional** — if user doesn't have one, set to `""` |
| PUBLIC_KEY | `==YourServerPublicKeyHere==` | The server's public key string. **Optional** — set to `""` to disable encryption |
| FIXED_PASSWORD | `YourPassword123` | Password for unattended access. **Optional** — set to `""` for no fixed password |

Default everything to empty strings if user doesn't specify, EXCEPT ID_SERVER and RELAY_SERVER which MUST be provided.

Ask: "Do you also need an API server URL (for user management) and a fixed password (for unattended access)?"

## Step 2: Check Git and GitHub Authentication

Before doing anything else, verify the environment is ready:

```powershell
# Check git config
git config --global user.name
git config --global user.email

# Check gh auth
gh auth status
```

**If gh is not logged in:** Tell the user they need to run `gh auth login` in their terminal, then tell you when it's done. Wait for them to confirm. Do NOT proceed without authentication.

**If git user.name/email is not set:** Tell the user to set them:
```powershell
git config --global user.name "YourName"
git config --global user.email "your@email.com"
```

## Step 3: Create the Workflow YAML

Read the template from `references/workflow-template.yml`. **Do NOT replace the `${{ secrets.XXX }}` placeholders** — they MUST remain as-is in the YAML file. The actual values will be stored in GitHub Secrets (Step 4.5).

The workflow reads secrets at runtime via `${{ secrets.ID_SERVER }}` etc. This means the YAML file contains **zero sensitive data** even though the repo is public.

## Step 4: Create Repository

Generate a date-based repo name:

```powershell
$date = Get-Date -Format "yyyyMMdd"
$repoName = "rustdesk-$date"
```

Create the repo as PUBLIC (required for free GitHub Actions):

```powershell
gh repo create $repoName --public --description "Custom RustDesk Windows build"
```

If the command fails with "already exists", append a counter: `rustdesk-20260722-2`, etc.

## Step 4.5: Store Config in GitHub Secrets (Security Critical)

This prevents sensitive data from being visible in the public repo. After creating the repo, use `gh secret set` to store each value:

```powershell
gh secret set ID_SERVER --repo $repoName --body "<id_server_value>"
gh secret set RELAY_SERVER --repo $repoName --body "<relay_server_value>"
gh secret set API_SERVER --repo $repoName --body "<api_server_value>"
gh secret set FIXED_PASSWORD --repo $repoName --body "<fixed_password_value>"
gh secret set PUBLIC_KEY --repo $repoName --body "<public_key_value>"
```

GitHub Secrets are encrypted and:
- Never visible in the repository UI or file content
- Automatically masked in Actions logs (even if accidentally printed)
- Only accessible to Actions runners at runtime
- The `${{ secrets.XXX }}` references in the YAML file are expanded by GitHub at runtime, not stored in the repo

**Do NOT skip this step.** Without it, the workflow will fail because the secret references in the YAML will be empty.

## Step 5: Initialize Git and Push

```powershell
# Initialize repo
git init
git branch -m main

# Create directory structure
New-Item -ItemType Directory -Path ".github\workflows" -Force

# Write the workflow YAML
# (Use the template with placeholders replaced)
```

Write the workflow file to `.github\workflows\build-rustdesk-windows.yml`.

```powershell
# Commit and push
git add -A
git commit -m "init: custom RustDesk build"
git remote add origin "https://github.com/<username>/$repoName.git"
git push -u origin main
```

## Step 6: Trigger Build

```powershell
gh workflow run build-rustdesk-windows.yml --repo <username>/$repoName
```

Wait a few seconds, then verify:

```powershell
gh run list --repo <username>/$repoName --limit 1 --json name,status,conclusion,databaseId
```

Return the Actions URL to the user:

```powershell
gh run view --repo <username>/$repoName --json url
```

## Known Failure Modes

If the build fails, here's how to diagnose and fix:

### YAML parsing error (0-second failure)
- Check the workflow YAML for indentation issues in `run: |` blocks
- All lines in a YAML literal block scalar must have SAME or MORE indentation than the first line
- The `python3 << 'PYEOF'` here-doc approach avoids this; make sure it's used for the config patching step

### LLVM version conflict
- The windows-2022 runner has LLVM 20.x pre-installed
- The workflow uses `KyleMayes/install-llvm-action@v2.0.9` which handles this
- Do NOT change to `choco install llvm`

### vcpkg ffmpeg headers missing
- Symptom: `fatal error C1083: Cannot open include file: 'libavutil/pixfmt.h'`
- Fix confirmed: The `rm -rf "$VCPKG_ROOT/installed/x64-windows-static"` before `vcpkg install` is critical
- Also ensure `VCPKG_BINARY_SOURCES` and `VCPKG_DEFAULT_HOST_TRIPLET` env vars are set

### Bridge generation fails
- The flutter-rust-bridge codegen MUST run on Linux (ubuntu-22.04), NOT on Windows
- Must use Flutter 3.22.3 for bridge generation (not the 3.24.5 used for Windows build)
- The `sed` line `s/extended_text: 14.0.0/extended_text: 13.0.0/g` is required for Flutter 3.22.3 compat

### `option_env!("FIXED_PASSWORD")` not found
- This error means the injected Rust code references a function that doesn't exist
- The `load_marker` string must EXACTLY match `fn load() -> Config {\n        let mut config = Config::load_::<Config>("");`
- If upstream changes this signature, update the marker string
- Do NOT use a separate `default_password()` function — inject the check inline

### Private repo billing
- Free GitHub accounts cannot run Actions on private repos
- Always create repo as PUBLIC
- If user later wants it private, they can change after the artifact is downloaded

## Important: Upstream Verification

Before writing the workflow, verify the current upstream config.rs patterns match. Use:

```bash
gh api repos/rustdesk/hbb_common/contents/src/config.rs | jq -r '.content' | base64 -d | grep -E "PROD_RENDEZVOUS_SERVER|DEFAULT_SETTINGS|RS_PUB_KEY"
```

If the patterns differ from what's in the template, **update the Python regex patterns** in the workflow template accordingly. The three critical patterns:

```python
# Pattern 1: PROD_RENDEZVOUS_SERVER
r'pub static ref PROD_RENDEZVOUS_SERVER: RwLock<String> = RwLock::new\(""\.to_owned\(\)\);'

# Pattern 2: DEFAULT_SETTINGS
r'pub static ref DEFAULT_SETTINGS: RwLock<HashMap<String, String>> = Default::default\(\);'

# Pattern 3: RS_PUB_KEY
r'pub const RS_PUB_KEY: &str = ".*?";'

# Pattern 4: Config::load() marker (must match exactly)
'fn load() -> Config {\n        let mut config = Config::load_::<Config>("");'
```

Also verify the `fn load() -> Config` marker hasn't changed:

```bash
gh api repos/rustdesk/hbb_common/contents/src/config.rs | jq -r '.content' | base64 -d | grep -A1 "^fn load()"
```
