# Agent Sandbox (Blank Slate)

Run Claude Code in a secure, sandboxed Docker container with network restrictions and filesystem isolation. Based on [Anthropic's reference devcontainer](https://github.com/anthropics/claude-code/tree/main/.devcontainer), modified for AWS Bedrock authentication and tighter security.

This is the **blank-slate** branch: a minimal container with just Claude Code and core tooling, intended for fast first builds and general-purpose use. If you need Python and R pre-installed (with `tidyverse` and `languageserver`), switch to the `main` branch instead.

## Getting Started

### Prerequisites

- [Docker Desktop](https://www.docker.com) (running)
- [VS Code](https://code.visualstudio.com) with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension

### Setup

The container supports two authentication modes. Pick one.

#### Option A: Claude.ai Max (recommended for individuals)

If you have a Claude.ai subscription (Pro or Max), no environment variables are required. Claude Code will launch an OAuth flow the first time you run `/login` inside the container.

Skip directly to step 2 below.

#### Option B: AWS Bedrock (for teams using AWS-managed billing)

Set your Bedrock credentials as environment variables before launching VS Code.

**macOS:** Add to `~/.zprofile`:
```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
export AWS_BEARER_TOKEN_BEDROCK=your-bedrock-api-key
export ANTHROPIC_DEFAULT_OPUS_MODEL=us.anthropic.claude-opus-4-6-v1
```

**Windows:** Open PowerShell as Administrator and run:
```powershell
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_USE_BEDROCK', '1', 'User')
[System.Environment]::SetEnvironmentVariable('AWS_REGION', 'us-east-1', 'User')
[System.Environment]::SetEnvironmentVariable('AWS_BEARER_TOKEN_BEDROCK', 'your-bedrock-api-key', 'User')
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_DEFAULT_OPUS_MODEL', 'us.anthropic.claude-opus-4-6-v1', 'User')
```

#### Launch the container

1. (Bedrock only) Open a **new terminal** after setting the env vars so VS Code picks them up.

2. Launch VS Code from the project directory:
   ```bash
   code /path/to/agent-sandbox
   ```

   > **macOS:** If you get `zsh: command not found: code`, open VS Code, press `Cmd+Shift+P`, type "shell command", and select **Shell Command: Install 'code' command in PATH**. Then retry.
   >
   > **Windows:** If `code` is not recognized, restart your terminal. If that doesn't help, reinstall VS Code with the "Add to PATH" option checked.

3. When VS Code prompts **"Reopen in Container?"**, click Yes. First build takes 2-5 minutes. Subsequent rebuilds are faster if Docker caches the layers. A "Dev Containers" log terminal will appear with setup output. Once it finishes, close that terminal tab and open a new one with `` Ctrl+` ``.

4. Type `claude`.

5. **If using Claude.ai Max (Option A):** run `/login` inside Claude and follow the OAuth prompt. Your credentials are stored in a named Docker volume, so you only need to log in once per sandbox.

6. Configure the Claude Code settings based on your preferences. Once you're in the Claude prompt, ensure you select the correct model using `/model` (e.g., `Opus 4.6`).

### Included Tooling

This branch ships the minimum needed to run Claude Code: Node 20, `git`, `gh`, `zsh` with powerlevel10k, `fzf`, `git-delta`, and the firewall hardening utilities (`iptables`, `ipset`, `dnsutils`, `jq`, `aggregate`). No language runtimes beyond Node are pre-installed. Add what you need to `.devcontainer/Dockerfile` and rebuild.

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

To start a new project, duplicate the entire `agent-sandbox` folder and rename it (e.g., `project-a`). Each copy runs its own isolated container with its own `workspace/`.

### Security

Claude's file access is restricted to `/workspace` via permission deny rules in `.claude/settings.json`. The container also blocks `sudo`, `chmod`, `chown`, and destructive `rm` commands.

## Differences from Anthropic's Reference

| Change | File | Why |
|---|---|---|
| Added `-exist` flag to `ipset add` calls | `init-firewall.sh` | Anthropic's script crashes when DNS returns duplicate IPs (e.g., `marketplace.visualstudio.com`). The `-exist` flag silently skips duplicates instead of failing. |
| Added AWS Bedrock endpoints to firewall allowlist | `init-firewall.sh` | The default firewall only allows `api.anthropic.com`. Bedrock needs `bedrock-runtime.us-east-1.amazonaws.com` and `bedrock.us-east-1.amazonaws.com`. |
| Added Claude.ai OAuth endpoints to firewall allowlist | `init-firewall.sh` | `claude.ai` and `console.anthropic.com` are required for the `/login` OAuth flow used by Claude.ai Pro/Max accounts. |
| Removed outbound SSH | `init-firewall.sh` | SSH (port 22) was allowed by default. Removed since it's unnecessary for research workloads and reduces attack surface. |
| Optional Bedrock auth via `containerEnv` | `devcontainer.json` | Passes `CLAUDE_CODE_USE_BEDROCK`, `AWS_BEARER_TOKEN_BEDROCK`, `AWS_REGION`, and `ANTHROPIC_DEFAULT_OPUS_MODEL` into the container via `${localEnv:...:}` with empty defaults. Unset on the host means the container boots for Claude.ai Max OAuth users; set on the host enables Bedrock mode. No secrets in the repo. |
| Mount only `workspace/` subdirectory | `devcontainer.json` | Anthropic's config mounts the entire project directory. This means Claude could modify `.devcontainer/` config files (firewall rules, Dockerfile, etc.). We mount only `workspace/` so Claude cannot access or alter the container configuration. |
| Disabled telemetry and auto-updates | `devcontainer.json` | VS Code telemetry and extension update checks are blocked by the firewall, causing noisy errors. Disabling them avoids the error spam. |
| Stop container on close | `devcontainer.json` | Prevents the container from running in the background after VS Code is closed. |
| Added permission deny rules | `workspace/.claude/settings.json` | Blocks Claude's Read/Edit tools from accessing system directories, and blocks `sudo`, `chmod`, `chown`, and destructive `rm` via Bash. |
| Retry transient network failures in firewall script | `init-firewall.sh` | The original script bails on the first failed `curl`/`dig`. Retries with backoff absorb DNS hiccups at container start. |
