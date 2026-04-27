# Agent Sandbox

Run Claude Code (and other coding agents) in a secure, sandboxed Docker container with network restrictions and filesystem isolation. Based on [Anthropic's reference devcontainer](https://github.com/anthropics/claude-code/tree/main/.devcontainer), modified for AWS Bedrock authentication and tighter security.

## Quick Start

The recommended way to scaffold a new sandbox is the [`@yale-dissc/create-agent-sandbox`](https://www.npmjs.com/package/@yale-dissc/create-agent-sandbox) wizard:

```bash
npm create @yale-dissc/agent-sandbox@latest my-research-project
```

Requires Node.js 18+. No global install is needed; `npm create` runs the latest version on demand.

The wizard:

1. Detects your installed tools (Git, Docker, VS Code, the Dev Containers extension, the GitHub CLI). System software (Git, Docker, VS Code, Node.js) is never installed automatically; the wizard opens the official download page in your browser instead. With your confirmation, the wizard will install the Dev Containers VS Code extension via the `code` CLI and start Docker Desktop if it is installed but not running.
2. Walks you through Claude.ai or AWS Bedrock authentication. For Bedrock, it writes the env vars to `~/.zprofile` (macOS) or User-scope env vars (Windows) with a timestamped backup and sentinel-bracketed block so future runs can update in place.
3. Asks which languages to pre-install in the container (Python 3, R + tidyverse, both, or neither — the default is neither, for the fastest first build) and edits the fetched `Dockerfile` accordingly.
4. Optionally runs `git init` in `workspace/` and asks for repository visibility (private, internal, or public) before creating the GitHub repo via `gh`.
5. Launches VS Code on the new project so you can click "Reopen in Container".

Useful flags:

- `npx @yale-dissc/create-agent-sandbox --check` audits your machine and exits without making changes.
- `npx @yale-dissc/create-agent-sandbox my-project --dry-run` prints every action that would be taken.
- `--ref <git-ref>` pins the template to a specific tag, branch, or commit of this repo.

See the [wizard documentation](https://www.npmjs.com/package/@yale-dissc/create-agent-sandbox) for the full flag list and safety guarantees.

## Prerequisites

- [Git](https://git-scm.com) (already present on most Linux distros, and on macOS once Xcode Command Line Tools are installed)
- [Docker Desktop](https://www.docker.com) (running)
- [VS Code](https://code.visualstudio.com) with the [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension
- Node.js 18+ (only needed for the wizard)
- [GitHub CLI](https://cli.github.com) (optional; only needed if you want the wizard to create a remote repo for you)

The wizard detects all of the above and opens the appropriate download page for anything missing, so you don't need to install them in advance.

## Daily Use

- **Start**: Open the project in VS Code, click "Reopen in Container", run `claude`.
- **Stop**: Close VS Code. Your files are on your host machine, nothing to save.
- **Something broke**: `Cmd/Ctrl+Shift+P` > "Rebuild Container Without Cache".
- **Add files**: Put them in `workspace/`, the only folder Claude can see.

Once you're in the Claude prompt, ensure you select the correct model with `/model` (e.g., `Opus 4.6`).

## Skipping Permission Prompts

Because the container is sandboxed (restricted network, isolated filesystem), you can safely run:

```bash
claude --dangerously-skip-permissions
```

This skips all tool approval prompts, so Claude can work faster without pausing for confirmation. The trade-off: Claude can modify or delete anything in `workspace/` without asking.

**Use Git inside `workspace/`.** Initialize a repo, commit your work frequently, and push to a remote. This gives you a full history of every change Claude makes and lets you revert anything unwanted. Without version control, there is no undo. The wizard does this for you on first run.

## Multiple Projects

To start a new project, run the wizard again with a different name:

```bash
npm create @yale-dissc/agent-sandbox@latest project-b
```

Each project lives in its own folder with its own `workspace/`, dev container, and (optionally) git history. Do not put multiple projects in the same `workspace/` folder; Claude operates on the entire workspace, so mixing projects risks unintended modifications across unrelated files.

## Security

Claude's file access is restricted to `/workspace` via permission deny rules in `.claude/settings.json`. The container also blocks `sudo`, `chmod`, `chown`, and destructive `rm` commands.

## Manual Setup (without the wizard)

If you prefer to clone the repo directly, you can. The wizard exists to automate the steps below for non-technical users; nothing about the repo requires it.

### Clone

```bash
git clone https://github.com/DISSC-yale/agent-sandbox.git my-project
cd my-project
```

If you want a minimal container with no Python or R pre-installed (faster first build), see [Removing R or Python from `main`](#removing-r-or-python-from-main) below before launching the container.

### Configure Authentication

The container supports two authentication modes. Pick one.

**Claude.ai Pro/Max (recommended for individuals).** No environment variables required. Claude Code launches an OAuth flow the first time you run `/login` inside the container. Skip to "Launch the Container" below.

**AWS Bedrock (for teams using AWS-managed billing).** Set your Bedrock credentials as environment variables before launching VS Code.

macOS, add to `~/.zprofile`:

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
export AWS_BEARER_TOKEN_BEDROCK=your-bedrock-api-key
export ANTHROPIC_DEFAULT_OPUS_MODEL=us.anthropic.claude-opus-4-6-v1
```

Windows, open PowerShell and run (User scope; no admin required):

```powershell
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_USE_BEDROCK', '1', 'User')
[System.Environment]::SetEnvironmentVariable('AWS_REGION', 'us-east-1', 'User')
[System.Environment]::SetEnvironmentVariable('AWS_BEARER_TOKEN_BEDROCK', 'your-bedrock-api-key', 'User')
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_DEFAULT_OPUS_MODEL', 'us.anthropic.claude-opus-4-6-v1', 'User')
```

Open a new terminal (macOS) or restart VS Code (Windows) so the new env vars are picked up.

### Launch the Container

```bash
code /path/to/my-project
```

> **macOS:** If you get `zsh: command not found: code`, open VS Code, press `Cmd+Shift+P`, type "shell command", and select **Shell Command: Install 'code' command in PATH**. Then retry.
>
> **Windows:** If `code` is not recognized, restart your terminal. If that doesn't help, reinstall VS Code with the "Add to PATH" option checked.

When VS Code prompts **"Reopen in Container?"**, click Yes. The first build with Python and R included takes 20-40 minutes; without them (see [Removing R or Python from `main`](#removing-r-or-python-from-main)) it takes 2-5 minutes. Subsequent rebuilds are faster if Docker caches the layers.

Once the build finishes:

1. Open a terminal in VS Code (`` Ctrl+` ``) and run `claude`.
2. **If using Claude.ai Pro/Max**: run `/login` and follow the OAuth prompt. Credentials are stored in a named Docker volume, so you only need to log in once per sandbox.
3. **If using AWS Bedrock**: run `/model` to confirm Claude is using the Bedrock-backed Opus model.

### Removing R or Python from `main`

If you started from `main` and want to remove R or Python before the first build, edit `.devcontainer/Dockerfile`:

1. Delete the relevant lines from the `apt-get install` block:
   - Python: `python3 \`, `python3-pip \`, `python3-venv \`
   - R: `r-base \`, `r-base-dev \`
2. If removing R, also delete the `RUN R -e 'install.packages(...)'` line.

The `locales` package and locale configuration can stay; they're useful for any workload.

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
| Added Python 3 and R | `Dockerfile` | Pre-installs `python3`, `pip`, `venv`, `r-base`, `r-base-dev`, and the `tidyverse` and `languageserver` R packages for research workloads. |
| Added permission deny rules | `workspace/.claude/settings.json` | Blocks Claude's Read/Edit tools from accessing system directories, and blocks `sudo`, `chmod`, `chown`, and destructive `rm` via Bash. |
