# Local Copilot plugin install and usage

Issue: <https://github.com/jwayong/OpenViking/issues/39>

This guide installs and uses the OpenViking Copilot memory plugins from this repo or from local artifacts. It does not require the VS Code Marketplace or the public npm registry for the OpenViking plugin packages.

## What gets installed

| Target | Installed artifact | User-facing entry point |
| --- | --- | --- |
| GitHub Copilot CLI | local `@openviking/copilot-cli-memory` `.tgz` | `openviking-copilot-mcp` MCP server mounted in `~/.copilot/mcp-config.json` |
| VS Code Copilot Chat | local `openviking-copilot.vsix` | `@openviking` chat participant, `openviking_recall` language-model tool, and OpenViking settings |

## Prerequisites

1. macOS or Linux.
2. `git`, `jq`, and `curl` on `PATH`.
3. Node.js/npm. Node `>=22` is expected by the Copilot workspaces.
4. An OpenViking server URL plus any account/user/API-key values your server requires.
5. For CLI use: the agentic GitHub Copilot CLI executable named `copilot`.
6. For VS Code use: VS Code `>=1.99`, GitHub Copilot Chat enabled, and the `code` CLI on `PATH`.

## Step 1: start from a local checkout

```bash
cd /path/to/OpenViking
git checkout feature/copilot-memory-plugin-plan
```

If you are installing from a branch that has not landed on `main` yet, run the helper from your local checkout instead of using the raw GitHub one-liner.

## Step 2: run the setup helper

```bash
bash examples/copilot/setup-helper/install.sh
```

The helper defaults to `OPENVIKING_INSTALL_SOURCE=local` and is safe to re-run. It backs up changed config files with `.bak.YYYYMMDD-HHMMSS`.

When prompted:

1. Enter the OpenViking URL, API key, account, user, and agent id. The default agent id is `copilot-cli` for the CLI path.
2. Let the helper reuse or refresh the local repo checkout.
3. Choose `Y` for VS Code extension install if `code` is available and you want the VS Code plugin.
4. Choose `Y` for CLI package install to pack and install a local `.tgz`.
5. Accept or edit the Copilot CLI `mcp-config.json` path. The default is `${COPILOT_HOME:-$HOME/.copilot}/mcp-config.json`.
6. Choose whether to add the optional `copilot()` shell wrapper. The wrapper only flushes captures the model already made; it cannot capture turns the model never sent to `openviking_capture`.

The helper creates or reuses artifacts in `${OPENVIKING_HOME:-$HOME/.openviking}/copilot-artifacts` unless `OPENVIKING_COPILOT_ARTIFACT_DIR` is set.

## Step 3: validate the install

```bash
openviking-copilot-mcp --check
```

Expected result: a redacted config summary showing the resolved OpenViking URL, account, user, agent id, and enabled status.

For VS Code, also check that the extension is installed:

```bash
code --list-extensions | grep -i openviking
```

If you enabled the shell wrapper, reload your shell before using `copilot`:

```bash
source ~/.zshrc  # or ~/.bashrc
```

## Step 4: use memory from the Copilot CLI

1. Start the Copilot CLI from a project directory:

   ```bash
   copilot
   ```

2. Ask a question that may need remembered project context, for example:

   ```text
   Use OpenViking memory if relevant. What did we decide about the Copilot local install flow?
   ```

3. The model can call `openviking_recall` to retrieve a ranked `<openviking-context>` block before answering.

4. At the end of useful turns, the model is asked to call `openviking_capture` with `{ user, assistant }` so the turn can be stored. This is model-discretion based: recall is available, but capture is not guaranteed if the model declines or forgets to call the tool.

5. Optional debug checks outside the Copilot CLI:

   ```bash
   openviking-copilot-mcp --debug-recall="Copilot local install flow"
   ```

   ```bash
   mkdir -p "$HOME/.openviking"
   cat > "$HOME/.openviking/copilot-turn.json" <<'JSON'
   [
     {"role":"user","text":"Remember that this repo uses local .vsix and .tgz installs for Copilot plugins."},
     {"role":"assistant","text":"Noted. The local-first Copilot install avoids Marketplace and npm publishing."}
   ]
   JSON
   openviking-copilot-mcp --debug-capture="$HOME/.openviking/copilot-turn.json"
   ```

## Step 5: use memory from VS Code Copilot Chat

1. Open VS Code from a project directory:

   ```bash
   code /path/to/project
   ```

2. If the API key was not configured by `~/.openviking/ovcli.conf`, run the command palette action `OpenViking: Set API Key`. This stores the key in VS Code SecretStorage.

3. Open Copilot Chat and ask a normal question. The installed `openviking_recall` language-model tool can retrieve relevant OpenViking memory for default chat turns.

4. Use the participant directly when you want explicit OpenViking memory behavior:

   ```text
   @openviking /recall Copilot local install flow
   ```

   ```text
   @openviking /store This repo should use local .vsix and .tgz artifacts for Copilot plugin installs.
   ```

   ```text
   @openviking Explain the current local-first plugin install path.
   ```

5. Completed `@openviking` participant turns are captured into OpenViking when `openviking.autoCapture` is enabled. Stable VS Code 1.99 APIs do not expose default `@workspace` response events to extensions, so default-chat capture is not currently supported.

## Reuse prebuilt local artifacts

Build artifacts once and reuse them across machines:

```bash
OPENVIKING_COPILOT_ARTIFACT_DIR=$PWD/.openviking-artifacts \
  bash examples/copilot/setup-helper/install.sh
```

Then install from explicit local files:

```bash
OPENVIKING_COPILOT_VSIX=/path/to/openviking-copilot.vsix \
OPENVIKING_COPILOT_CLI_TGZ=/path/to/openviking-copilot-cli-memory-0.0.0.tgz \
  bash examples/copilot/setup-helper/install.sh
```

## Optional registry mode

Public publishing is tracked separately in #31 and #32. If a future private or public registry is available, opt into registry install explicitly:

```bash
OPENVIKING_INSTALL_SOURCE=registry bash examples/copilot/setup-helper/install.sh
```

## Private extension galleries

Standard Microsoft VS Code users should use `.vsix` install for local/private testing. Custom extension galleries require a host that supports overriding `extensionsGallery` (for example an Open VSX-compatible gallery in OSS/VSCodium-style builds), so this repo does not depend on that path for local installs.

## Troubleshooting

- `openviking-copilot-mcp --check` exits non-zero: verify `~/.openviking/ovcli.conf` or the `OPENVIKING_*` environment variables.
- `code --list-extensions` does not show OpenViking: confirm the `code` CLI is on `PATH`, then re-run the helper and accept VS Code installation.
- Copilot CLI does not show OpenViking tools: inspect `${COPILOT_HOME:-$HOME/.copilot}/mcp-config.json` and confirm it contains the `openviking` server entry with command `openviking-copilot-mcp`.
- CLI capture did not happen: ask the model to call `openviking_capture`, or use the optional shell wrapper to force-flush pending captured turns at process exit.
- VS Code default chat is not captured: use `@openviking` participant turns for capture until VS Code exposes a default-chat turn event.
