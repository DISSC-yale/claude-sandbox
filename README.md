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

3. When VS Code prompts **"Reopen in Container?"**, click Yes. First build takes 2-5 minutes. A "Dev Containers" log terminal will appear with setup output. Once it finishes, close that terminal tab and open a new one with `` Ctrl+` ``.

4. Type `claude`.

5. Configure the Claude Code settings based on your preferences. Once you're in the Claude prompt, ensure you select the correct model using `/model` (e.g., `Opus 4.6`).

### Included Languages

Python 3 and R are pre-installed in the container. Use `python3` and `R` from the terminal.

### Daily Use

- **Start**: Open the project in VS Code, click "Reopen in Container", run `claude`
- **Stop**: Close VS Code. Your files are on your Mac, nothing to save.
- **Something broke**: `Cmd+Shift+P` > "Rebuild Container Without Cache"
- **Add files**: Put them in `workspace/`, the only folder Claude can see

### Skipping Permission Prompts

Because the container is sandboxed (restricted network, isolated filesystem), you can safely run:

```bash
claude --dangerously-skip-permissions
```

This skips all tool approval prompts, so Claude can work faster without pausing for confirmation. The trade-off: Claude can modify or delete anything in `workspace/` without asking. Make sure any important files are backed up outside of the workspace before using this flag.

### Security

Claude's file access is restricted to `/workspace` via permission deny rules in `.claude/settings.json`. The container also blocks `sudo`, `chmod`, `chown`, and destructive `rm` commands.

## Differences from Anthropic's Reference

| Change | File | Why |
|---|---|---|
| Added `-exist` flag to `ipset add` calls | `init-firewall.sh` | Anthropic's script crashes when DNS returns duplicate IPs (e.g., `marketplace.visualstudio.com`). The `-exist` flag silently skips duplicates instead of failing. |
| Added AWS Bedrock endpoints to firewall allowlist | `init-firewall.sh` | The default firewall only allows `api.anthropic.com`. Bedrock needs `bedrock-runtime.us-east-1.amazonaws.com` and `bedrock.us-east-1.amazonaws.com`. |
| Removed outbound SSH | `init-firewall.sh` | SSH (port 22) was allowed by default. Removed since it's unnecessary for research workloads and reduces attack surface. |
| Added Bedrock auth to `containerEnv` | `devcontainer.json` | Passes `CLAUDE_CODE_USE_BEDROCK`, `AWS_BEARER_TOKEN_BEDROCK`, `AWS_REGION`, and a pinned Opus model into the container via `${localEnv:...}` (pulls from the user's Mac environment, no secrets in the repo). |
| Mount only `workspace/` subdirectory | `devcontainer.json` | Anthropic's config mounts the entire project directory. This means Claude could modify `.devcontainer/` config files (firewall rules, Dockerfile, etc.). We mount only `workspace/` so Claude cannot access or alter the container configuration. |
| Disabled telemetry and auto-updates | `devcontainer.json` | VS Code telemetry and extension update checks are blocked by the firewall, causing noisy errors. Disabling them avoids the error spam. |
| Stop container on close | `devcontainer.json` | Prevents the container from running in the background after VS Code is closed. |
| Added Python 3 and R | `Dockerfile` | Pre-installs `python3`, `pip`, `venv`, `r-base`, and `r-base-dev` for research workloads. |
| Added permission deny rules | `workspace/.claude/settings.json` | Blocks Claude's Read/Edit tools from accessing system directories, and blocks `sudo`, `chmod`, `chown`, and destructive `rm` via Bash. |
