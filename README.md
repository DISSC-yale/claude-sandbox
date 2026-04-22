# Claude Code Sandbox

Run Claude Code in a secure, sandboxed Docker container with network restrictions and filesystem isolation. Based on [Anthropic's reference devcontainer](https://github.com/anthropics/claude-code/tree/main/.devcontainer), modified for AWS Bedrock authentication and tighter security.

## Getting Started

### Prerequisites

- [Docker Desktop](https://www.docker.com) (running)
- [VS Code](https://code.visualstudio.com) with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension

### Setup

1. Set your Bedrock credentials as environment variables:

   **macOS:** Add to `~/.zprofile`:
   ```bash
   export CLAUDE_CODE_USE_BEDROCK=1
   export AWS_REGION=us-east-1
   export AWS_BEARER_TOKEN_BEDROCK=your-bedrock-api-key
   ```

   **Windows:** Open PowerShell as Administrator and run:
   ```powershell
   [System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_USE_BEDROCK', '1', 'User')
   [System.Environment]::SetEnvironmentVariable('AWS_REGION', 'us-east-1', 'User')
   [System.Environment]::SetEnvironmentVariable('AWS_BEARER_TOKEN_BEDROCK', 'your-bedrock-api-key', 'User')
   ```

2. Open a **new terminal** and launch VS Code from it (so it picks up the env vars):
   ```bash
   code /path/to/claude-sandbox
   ```

   > **macOS:** If you get `zsh: command not found: code`, open VS Code, press `Cmd+Shift+P`, type "shell command", and select **Shell Command: Install 'code' command in PATH**. Then retry.
   >
   > **Windows:** If `code` is not recognized, restart your terminal. If that doesn't help, reinstall VS Code with the "Add to PATH" option checked.

3. When VS Code prompts **"Reopen in Container?"**, click Yes. First build takes 20-40 minutes with R and Python included, or 2-5 minutes without (see [Included Languages](#included-languages) below). Subsequent rebuilds are faster if Docker caches the layers. A "Dev Containers" log terminal will appear with setup output. Once it finishes, close that terminal tab and open a new one with `` Ctrl+` ``.

4. Type `claude`.

5. Configure the Claude Code settings based on your preferences. Once you're in the Claude prompt, ensure you select the correct model using `/model` (e.g., `Opus 4.6`).

### Included Languages

Python 3 and R are pre-installed in the container, along with the `tidyverse` and `languageserver` R packages. Use `python3` and `R` from the terminal.

If you don't need R and Python, removing them before the first build shrinks the image and cuts build time from 20-40 minutes to 2-5 minutes:

1. In `.devcontainer/Dockerfile`, delete these lines from the `apt-get install` block:
   - `python3 \`
   - `python3-pip \`
   - `python3-venv \`
   - `r-base \`
   - `r-base-dev \`
2. Delete the `RUN R -e 'install.packages(...)'` line (if present)

The `locales` package and locale configuration can stay, they're useful for any workload.

### Daily Use

- **Start**: Open the project in VS Code, click "Reopen in Container", run `claude`
- **Stop**: Close VS Code. Your files are on your host machine, nothing to save.
- **Something broke**: `Cmd/Ctrl+Shift+P` > "Rebuild Container Without Cache"
- **Add files**: Put them in `workspace/`, the only folder Claude can see

### Skipping Permission Prompts

Because the container is sandboxed (restricted network, isolated filesystem), you can safely run:

```bash
claude --dangerously-skip-permissions
```

This skips all tool approval prompts, so Claude can work faster without pausing for confirmation. The trade-off: Claude can modify or delete anything in `workspace/` without asking.

**Use Git inside `workspace/`.** Initialize a repo, commit your work frequently, and push to a remote. This gives you a full history of every change Claude makes and lets you revert anything unwanted. Without version control, there is no undo.

### One Project Per Sandbox

Do not put multiple projects in the same `workspace/` folder. Claude operates on the entire workspace, so mixing projects risks unintended modifications across unrelated files.

To start a new project, duplicate the entire `claude-sandbox` folder and rename it (e.g., `project-a`). Each copy runs its own isolated container with its own `workspace/`.

### Security

Claude's file access is restricted to `/workspace` via permission deny rules in `.claude/settings.json`. The container also blocks `sudo`, `chmod`, `chown`, and destructive `rm` commands.

## Differences from Anthropic's Reference

| Change | File | Why |
|---|---|---|
| Added `-exist` flag to `ipset add` calls | `init-firewall.sh` | Anthropic's script crashes when DNS returns duplicate IPs (e.g., `marketplace.visualstudio.com`). The `-exist` flag silently skips duplicates instead of failing. |
| Added AWS Bedrock endpoints to firewall allowlist | `init-firewall.sh` | The default firewall only allows `api.anthropic.com`. Bedrock needs `bedrock-runtime.us-east-1.amazonaws.com` and `bedrock.us-east-1.amazonaws.com`. |
| Removed outbound SSH | `init-firewall.sh` | SSH (port 22) was allowed by default. Removed since it's unnecessary for research workloads and reduces attack surface. |
| Added Bedrock auth to `containerEnv` | `devcontainer.json` | Passes `CLAUDE_CODE_USE_BEDROCK`, `AWS_BEARER_TOKEN_BEDROCK`, `AWS_REGION`, and a pinned Opus model into the container via `${localEnv:...}` (pulls from the host environment, no secrets in the repo). |
| Mount only `workspace/` subdirectory | `devcontainer.json` | Anthropic's config mounts the entire project directory. This means Claude could modify `.devcontainer/` config files (firewall rules, Dockerfile, etc.). We mount only `workspace/` so Claude cannot access or alter the container configuration. |
| Disabled telemetry and auto-updates | `devcontainer.json` | VS Code telemetry and extension update checks are blocked by the firewall, causing noisy errors. Disabling them avoids the error spam. |
| Stop container on close | `devcontainer.json` | Prevents the container from running in the background after VS Code is closed. |
| Added Python 3 and R | `Dockerfile` | Pre-installs `python3`, `pip`, `venv`, `r-base`, `r-base-dev`, and the `tidyverse` and `languageserver` R packages for research workloads. |
| Added permission deny rules | `workspace/.claude/settings.json` | Blocks Claude's Read/Edit tools from accessing system directories, and blocks `sudo`, `chmod`, `chown`, and destructive `rm` via Bash. |
