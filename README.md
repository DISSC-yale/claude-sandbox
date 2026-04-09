# Claude Code Sandbox

Run Claude Code in a secure, sandboxed Docker container with network restrictions and filesystem isolation. Based on [Anthropic's reference devcontainer](https://github.com/anthropics/claude-code/tree/main/.devcontainer), modified for AWS Bedrock authentication and tighter security.

## Getting Started

### Prerequisites

- [Docker Desktop](https://www.docker.com) (running)
- [VS Code](https://code.visualstudio.com) with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension

### Setup

1. Add your Bedrock credentials to `~/.zprofile`:
   ```bash
   export CLAUDE_CODE_USE_BEDROCK=1
   export AWS_REGION=us-east-1
   export AWS_BEARER_TOKEN_BEDROCK=your-bedrock-api-key
   ```

2. Open a **new terminal** and launch VS Code from it (so it picks up the env vars):
   ```bash
   code /path/to/claude-sandbox
   ```

   > **Note:** If you get `zsh: command not found: code`, you need to install the `code` CLI. Open VS Code, press `Cmd+Shift+P`, type "shell command", and select **Shell Command: Install 'code' command in PATH**. Then retry the command above.

3. When VS Code prompts **"Reopen in Container?"**, click Yes. First build takes 2-5 minutes. Some code will scroll down the terminal and eventually say `Done. Press any key to close the terminal.` Press any key to continue.

4. Open the terminal (`` Ctrl+` ``) and type `claude`.

5. Configure the Claude Code settings based on your preferences. Once you're in the Claude prompt, ensure you select the correct model using `/model` (e.g., `Opus 4.6`).

### Daily Use

- **Start**: Open the project in VS Code, click "Reopen in Container", run `claude`
- **Stop**: Close VS Code. Your files are on your Mac, nothing to save.
- **Something broke**: `Cmd+Shift+P` > "Rebuild Container Without Cache"
- **Add files**: Put them in `workspace/`, the only folder Claude can see

## Differences from Anthropic's Reference

| Change | File | Why |
|---|---|---|
| Added `-exist` flag to `ipset add` calls | `init-firewall.sh` | Anthropic's script crashes when DNS returns duplicate IPs (e.g., `marketplace.visualstudio.com`). The `-exist` flag silently skips duplicates instead of failing. |
| Added AWS Bedrock endpoints to firewall allowlist | `init-firewall.sh` | The default firewall only allows `api.anthropic.com`. Bedrock needs `bedrock-runtime.us-east-1.amazonaws.com` and `bedrock.us-east-1.amazonaws.com`. |
| Added Bedrock auth to `containerEnv` | `devcontainer.json` | Passes `CLAUDE_CODE_USE_BEDROCK`, `AWS_BEARER_TOKEN_BEDROCK`, `AWS_REGION`, and a pinned Opus model into the container via `${localEnv:...}` (pulls from the user's Mac environment, no secrets in the repo). |
| Mount only `workspace/` subdirectory | `devcontainer.json` | Anthropic's config mounts the entire project directory. This means Claude could modify `.devcontainer/` config files (firewall rules, Dockerfile, etc.). We mount only `workspace/` so Claude cannot access or alter the container configuration. |
