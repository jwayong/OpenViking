# Install the OpenViking memory plugin for GitHub Copilot

A step-by-step guide. The same MCP server backs both the **GitHub Copilot CLI** and the **VS Code Copilot Chat** extension; pick whichever target you want and follow the matching steps.

## What you'll end up with

| Target | Installed artifact | Where it lives |
| --- | --- | --- |
| GitHub Copilot CLI | npm-global bin `openviking-copilot-mcp` | mounted via `~/.copilot/mcp-config.json` |
| VS Code Copilot Chat | `openviking-copilot.vsix` | `@openviking` chat participant + `openviking_recall` language-model tool |

Both targets read OpenViking server settings from `~/.openviking/ovcli.conf`.

## Prerequisites

- macOS or Linux
- `git`, `jq`, `curl` on `PATH`
- Node.js >= 22 with `npm`
- A reachable OpenViking server (local self-hosted, or Volcengine OpenViking Cloud + API key)
- For CLI use: the agentic `copilot` CLI installed and logged in (`npm i -g @github/copilot && copilot`)
- For VS Code use: VS Code >= 1.99 with GitHub Copilot Chat enabled, and the `code` CLI on `PATH`

## Quick path: run the installer

From a checkout of this repo:

```bash
cd /path/to/OpenViking
bash examples/copilot/setup-helper/install.sh
```

The installer auto-detects that it's running from a local checkout and uses that source — no clone needed. It is safe to re-run; existing config files are backed up as `<file>.bak.YYYYMMDD-HHMMSS`.

What the installer does, in order:

1. Checks OS + required CLIs (`git`, `jq`, `curl`, optionally `npm`/`code`).
2. Creates or reuses `~/.openviking/ovcli.conf` (URL, API key, account, user, agent id).
3. Resolves the source repo (local checkout > `OPENVIKING_REPO_DIR` > clone of `volcengine/OpenViking`).
4. Optional: packages a `.vsix` and installs the VS Code extension.
5. Packs `@openviking/copilot-cli-memory` into a local `.tgz`, runs `npm i -g <tgz>`, resolves the **absolute path** of the installed `openviking-copilot-mcp` binary, runs `--check` to verify, then merges that absolute path into `~/.copilot/mcp-config.json`.
6. Optional: appends a `copilot()` shell wrapper that flushes pending captures at end-of-session.

If the binary install fails for any reason, the installer **does not** write a broken `mcp-config.json` entry — it warns and skips the merge. Re-run after fixing the install.

Skip ahead to [Step 6: validate](#step-6-validate-the-install) when the script finishes, or follow the manual steps below.

---

## Manual install — Copilot CLI (MCP server)

### Step 1: configure OpenViking

```bash
mkdir -p ~/.openviking && chmod 700 ~/.openviking
cat > ~/.openviking/ovcli.conf <<'JSON'
{
  "url": "http://127.0.0.1:1933",
  "api_key": "",
  "account": "",
  "user": "",
  "agent_id": "copilot-cli"
}
JSON
chmod 600 ~/.openviking/ovcli.conf
```

For Volcengine OpenViking Cloud, set `url` to `https://api.vikingdb.cn-beijing.volces.com/openviking` and put your API key in `api_key`.

### Step 2: build and install the MCP server

From this repo:

```bash
cd examples/copilot
npm install
npm pack -w @openviking/copilot-cli-memory --pack-destination /tmp
npm i -g /tmp/openviking-copilot-cli-memory-*.tgz
```

Verify the bin landed on `PATH` and the config is wired up:

```bash
which openviking-copilot-mcp
openviking-copilot-mcp --check
```

`--check` should exit `0` and print `enabled : true` along with a redacted config summary.

### Step 3: register it with the Copilot CLI

Resolve the absolute path of the bin and write it into `~/.copilot/mcp-config.json`. Using the absolute path (not just `openviking-copilot-mcp`) avoids `PATH` issues when the Copilot CLI spawns the server from a non-login shell.

```bash
mkdir -p ~/.copilot
BIN=$(command -v openviking-copilot-mcp)
[ -x "$BIN" ] || { echo "openviking-copilot-mcp not on PATH"; exit 1; }

tmp=$(mktemp)
jq --arg bin "$BIN" --arg conf "$HOME/.openviking/ovcli.conf" '
  .mcpServers = (.mcpServers // {}) |
  .mcpServers.openviking = {
    type: "local",
    command: $bin,
    args: [],
    env: {
      OPENVIKING_MEMORY_ENABLED: "true",
      OPENVIKING_CLI_CONFIG_FILE: $conf
    },
    tools: ["*"]
  }
' "${HOME}/.copilot/mcp-config.json" 2>/dev/null \
  || jq --arg bin "$BIN" --arg conf "$HOME/.openviking/ovcli.conf" -n '
    {mcpServers: {openviking: {
      type: "local",
      command: $bin,
      args: [],
      env: {OPENVIKING_MEMORY_ENABLED: "true", OPENVIKING_CLI_CONFIG_FILE: $conf},
      tools: ["*"]
    }}}' \
  > "$tmp"
mv "$tmp" ~/.copilot/mcp-config.json
```

The resulting entry looks like:

```json
{
  "mcpServers": {
    "openviking": {
      "type": "local",
      "command": "/Users/you/.npm-global/bin/openviking-copilot-mcp",
      "args": [],
      "env": {
        "OPENVIKING_MEMORY_ENABLED": "true",
        "OPENVIKING_CLI_CONFIG_FILE": "/Users/you/.openviking/ovcli.conf"
      },
      "tools": ["*"]
    }
  }
}
```

### Step 4: use it

```bash
copilot
```

Inside Copilot CLI, ask a question; the model can call the `openviking_recall` MCP tool to retrieve a ranked `<openviking-context>` block, and `openviking_capture` with `{user, assistant}` to commit useful turns. Capture is model-discretion based — recall always works, capture happens when the model decides to call the tool.

---

## Manual install — VS Code extension

### Step 1: configure OpenViking

Same as Step 1 above. The extension reads the same `~/.openviking/ovcli.conf`. If you'd rather not put the API key in the file, leave `api_key` empty and run `OpenViking: Set API Key` from the VS Code command palette to store it in SecretStorage.

### Step 2: package the extension

```bash
cd examples/copilot
npm install
npm run build -w openviking-copilot
cd vscode-extension
npx vsce package --no-dependencies --skip-license --out openviking-copilot.vsix
```

### Step 3: install it

```bash
code --install-extension openviking-copilot.vsix --force
code --list-extensions | grep -i openviking   # sanity check
```

### Step 4: use it

Open VS Code on a project, open Copilot Chat, then either ask a normal question (the `openviking_recall` language-model tool can fire automatically) or address the participant directly:

```text
@openviking /recall Copilot install flow
@openviking /store This repo uses local .vsix and .tgz artifacts.
@openviking Explain the local-first plugin install path.
```

Completed `@openviking` participant turns are captured to OpenViking when `openviking.autoCapture` is enabled. Default `@workspace` turn capture isn't supported — VS Code 1.99 doesn't expose a turn-level event for default chat.

---

## Step 6: validate the install

```bash
openviking-copilot-mcp --check
```

Expect `enabled : true` and a redacted summary showing `baseUrl`, `agentId`, etc.

```bash
cat ~/.copilot/mcp-config.json | jq '.mcpServers.openviking'
```

Expect `command` to be an absolute path to a real file, and `env.OPENVIKING_CLI_CONFIG_FILE` to point at your `ovcli.conf`.

For the VS Code extension:

```bash
code --list-extensions | grep -i openviking
```

---

## Optional: copilot() shell wrapper

The installer can append a `copilot()` zsh/bash function that runs the regular `copilot` CLI and then calls `openviking-copilot-mcp --commit-flush` at exit, so any pending captures the model made via `openviking_capture` get committed even if no token-threshold trigger fired during the session. This only flushes captures the model already requested — it cannot capture turns the model never sent to the tool.

To enable later, re-run the installer and answer **Y** at the wrapper prompt, or `source examples/copilot/cli-plugin/wrapper/copilot.sh` from your shell rc.

## Reusable artifacts

```bash
OPENVIKING_COPILOT_ARTIFACT_DIR=$PWD/.openviking-artifacts \
  bash examples/copilot/setup-helper/install.sh
```

Then on another machine:

```bash
OPENVIKING_COPILOT_VSIX=/path/to/openviking-copilot.vsix \
OPENVIKING_COPILOT_CLI_TGZ=/path/to/openviking-copilot-cli-memory-0.0.0.tgz \
  bash examples/copilot/setup-helper/install.sh
```

## Troubleshooting

- **`openviking-copilot-mcp: command not found`** — `npm i -g <tgz>` didn't put the bin in your `PATH`. Check `npm prefix -g` and ensure its `bin/` is on `PATH` (commonly `~/.npm-global/bin`).
- **Copilot CLI doesn't list OpenViking tools** — open `~/.copilot/mcp-config.json` and confirm `mcpServers.openviking.command` points at an existing executable. The installer writes an absolute path; a relative `openviking-copilot-mcp` only works if the Copilot CLI's spawn environment also has the bin on `PATH`.
- **`--check` exits non-zero** — `enabled` is `false`. Verify `~/.openviking/ovcli.conf` (or `OPENVIKING_*` env vars) and that `OPENVIKING_MEMORY_ENABLED` is not explicitly `false`.
- **Installer says `Copilot example missing at ...`** — your source checkout doesn't include `examples/copilot/cli-plugin/`. Run from a checkout that does, or set `OPENVIKING_REPO_DIR` to one. The default clone is `volcengine/OpenViking` `main`; until this PR lands there, use a local checkout.
- **VS Code default-chat turns don't get captured** — expected. Use `@openviking` participant turns for capture, or rely on recall-only for default chat.
- **CLI capture didn't happen for a turn** — capture is model-discretion. Ask the model to call `openviking_capture`, or enable the `copilot()` shell wrapper to flush pending captures at exit.
