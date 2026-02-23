# Claude Code Devcontainer Template

A devcontainer template for running Claude Code with defence-in-depth against prompt injection and data exfiltration.

## Threat model

LLMs cannot sanitise their inputs. Malicious instructions hidden in code, READMEs, issues, or any content Claude reads can manipulate its behaviour. This is fundamentally different from traditional access control, where the entity taking actions can be trusted to act on its own intent.

This template mitigates two specific threats:

- **Prompt injection** -- malicious instructions in repo content that manipulate Claude's behaviour
- **Data exfiltration** -- Claude being tricked into sending your code or context to an attacker via git push, gist creation, curl, or other network tools

## Security layers

No single layer is sufficient. The template uses multiple independent controls:

### 1. Network firewall (iptables)

`init-firewall.sh` sets a default-DROP policy and only allows outbound connections to:

- GitHub (IP ranges from the GitHub API)
- npm registry
- Anthropic APIs
- VS Code marketplace and extensions
- Localhost and host network

All other outbound traffic is rejected. This runs at container start via `postStartCommand`.

### 2. Bubblewrap sandbox

Claude Code's sandbox (`settings.local.json`) wraps most bash commands in bubblewrap, which restricts filesystem and network access at the process level. Key settings:

- `enabled: true` -- sandbox is active
- `autoAllowBashIfSandboxed: true` -- sandboxed commands don't need manual approval
- `excludedCommands: ["git", "gh"]` -- git and gh bypass the sandbox (because bwrap blocks them entirely in unprivileged containers), secured by the firewall, pre-push hook, and permission deny list instead
- `enableWeakerNestedSandbox: true` -- required for running inside Docker without privileged mode

**Tradeoff**: git must be excluded from the sandbox because bwrap cannot create namespaces in unprivileged containers. This is acceptable because the iptables firewall restricts git's network access, and the pre-push hook restricts which remotes it can push to.

### 3. Pre-push hook (.githooks/pre-push)

Only allows `git push` to repos owned by a configured GitHub user. Even if Claude is tricked into pushing to an attacker's repo, git itself blocks it.

Setup after cloning:
```bash
git config core.hooksPath .githooks
git config --local hooks.allowedGithubUser <your-github-username>
```

**Tradeoff**: Claude can still fetch/clone from any GitHub repo (the firewall allows all GitHub IPs). This is necessary for installing dependencies and reading issues, but means injected content from public repos remains a vector for prompt injection. The defence here is Claude's own instruction-following, not a technical control.

### 4. Permission deny list

`settings.local.json` blocks Claude from:

- **`gh gist`** -- prevents gist creation as an exfiltration path
- **Sensitive files** -- `~/.ssh/`, `~/.gnupg/`, `~/.aws/`, `.env*`, etc.
- **Shell config** -- `~/.bashrc`, `~/.zshrc`, `~/.profile`

**Tradeoff**: `gh` is kept installed (not removed) because `gh issue` and `gh pr` are valuable for development workflows. The `gh gist` deny rule blocks the obvious exfiltration command, and bwrap blocks `curl`/`wget` as alternatives. However, a sufficiently creative injection could potentially use other tools or encoding tricks -- this is a probabilistic defence, not an absolute one.

### 5. GitHub token (external env file)

`gh` is authenticated via a `GH_TOKEN` environment variable, injected from a file on the host that is **not mounted** into the container. This means Claude cannot read the token file, even though the token is available in the container's environment (which `gh` needs).

Create the env file on the host:
```bash
mkdir -p ~/.config/<project-name>
echo "GH_TOKEN=ghp_yourtoken" > ~/.config/<project-name>/.env
```

The `devcontainer.json` references it via `--env-file`:
```json
"runArgs": [
    "--env-file", "${localEnv:HOME}/.config/<project-name>/.env",
    ...
]
```

Docker reads the file on the host at container creation and sets the env var. The file itself is never inside the container. Use a fine-grained PAT scoped to specific repos with only Issues and Pull Requests read/write permissions.

**Tradeoff**: Claude can still read the token value from the container environment at runtime (e.g. `echo $GH_TOKEN`). This is unavoidable if `gh` needs it. The mitigations are: the PAT is narrowly scoped, `gh gist` is denied, and bwrap blocks `curl`/`wget` so the token can't easily be sent elsewhere.

### 6. SSH agent forwarding

VS Code Dev Containers automatically forwards the host's SSH agent. The private key never enters the container -- only the agent socket is shared. Claude can authenticate git operations but cannot read the key material.

Host setup:
```bash
# Add to ~/.bashrc on the host
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
if ! ssh-add -l &>/dev/null; then
    rm -f "$SSH_AUTH_SOCK"
    eval "$(ssh-agent -a "$SSH_AUTH_SOCK" -s)" > /dev/null
    ssh-add ~/.ssh/id_ed25519
fi
```

Launch VS Code from a shell where the agent is running.

## What isn't covered

- **Prompt injection via fetched content** -- Claude can still read malicious instructions from any GitHub repo it clones or any issue it reads. No technical control prevents this; it relies on Claude's own resistance to injection.
- **Exfiltration via DNS** -- data could theoretically be encoded in DNS queries. The firewall allows DNS (port 53) for domain resolution.
- **Exfiltration via commit messages or branch names** -- pushing to an allowed repo with data encoded in metadata.
- **Container escape** -- this template assumes the container itself is trusted. If an attacker escapes the container, all bets are off.

## Setup

1. Copy the `.devcontainer/`, `.claude/`, and `.githooks/` directories into your project
2. Create the GitHub token env file on the host:
   ```bash
   mkdir -p ~/.config/<project-name>
   echo "GH_TOKEN=ghp_yourtoken" > ~/.config/<project-name>/.env
   ```
3. Update the `--env-file` path in `.devcontainer/devcontainer.json` to match
4. Open in VS Code and reopen in container
5. Configure the pre-push hook:
   ```bash
   git config core.hooksPath .githooks
   git config --local hooks.allowedGithubUser <your-github-username>
   ```
6. Ensure SSH agent is running on your host before launching VS Code
