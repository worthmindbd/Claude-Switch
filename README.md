# claude-switch

Switch [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) between the **Z.ai GLM API** and your **Claude subscription** with one command.

```bash
claude-switch glm       # use GLM models (+ Z.ai MCP servers)
claude-switch claude    # use your Claude subscription
claude-switch status    # show the current mode
```

The GLM setup follows Z.ai's official Claude Code configuration (<https://docs.z.ai/devpack/tool/claude>): an `env` block in `~/.claude/settings.json` plus four Z.ai MCP servers (web search, web reader, Zread, vision).

## Requirements

- Linux or macOS with `bash`
- [`jq`](https://jqlang.org/) (`apt install jq`, `dnf install jq`, `pacman -S jq`, `brew install jq`)
- Claude Code installed (`npm install -g @anthropic-ai/claude-code`). The `claude` command is needed for the MCP part.
- A Z.ai **GLM Coding Plan** API key (see [Which key to use](#which-key-to-use))

## Install

```bash
install -m 755 claude-switch ~/.local/bin/claude-switch
```

Make sure `~/.local/bin` is on your `PATH`. Run the install from a folder that is **not** synced (see [Security](#security)).

## Usage

| Command | What it does |
| --- | --- |
| `claude-switch glm` | Writes the GLM env block into `settings.json` and registers the Z.ai MCP servers |
| `claude-switch claude` | Saves the GLM block, removes it from `settings.json`, and unregisters the Z.ai MCP servers |
| `claude-switch status` | Shows mode, endpoint, models, key (last 4 characters), MCP servers registered, saved files |

### Options

| Option | Applies to | Effect |
| --- | --- | --- |
| `--new-key` | `glm` | Re-enter the Z.ai API key. Settings and MCP servers are re-registered with the new key. |
| `--no-mcp` | `glm`, `claude` | Leave the MCP servers exactly as they are |
| `--force` | `glm`, `claude` | Also act when `settings.json` points at a non-Z.ai `ANTHROPIC_BASE_URL` (otherwise the script refuses) |

**Restart Claude Code after every switch.** A running session keeps its old config, and MCP servers load only at startup.

## First-time setup (fresh install)

1. Install Claude Code and this script.
2. Run `claude-switch glm`.
3. It asks for your Z.ai API key (hidden input). To skip the prompt, export it first:
   ```bash
   export ZAI_API_KEY=your_key
   claude-switch glm
   ```
4. Start `claude`, then check with `/status` (model should be `glm-5.3`) and `/mcp` (four Z.ai servers).

You only enter the key once. Later switches reuse the saved key.

To use your Claude subscription instead, run `claude`, then `/login`. `claude-switch claude` does nothing on a fresh install because there is no GLM block to remove.

## What gets changed

**GLM mode** sets these keys under `env` in `~/.claude/settings.json` (Z.ai's official template):

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "your_zai_api_key",
    "ANTHROPIC_BASE_URL": "https://api.z.ai/api/anthropic",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-5.3-flash[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5.3[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5.3[1m]",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "1000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": 1,
    "API_TIMEOUT_MS": "3000000"
  }
}
```

**MCP servers** (user scope), managed through the `claude mcp` command:

| Name | Type | Tools |
| --- | --- | --- |
| `web-search-prime` | HTTP | `webSearchPrime` |
| `web-reader` | HTTP | `webReader` |
| `zread` | HTTP | `search_doc`, `get_repo_structure`, `read_file` |
| `zai-mcp-server` | local (`npx`) | vision tools; needs Node.js 22+ |

**Claude mode** removes only those eight `env` keys and those four MCP servers. Every other setting and every other MCP server is left alone.

## Files

| Path | Purpose |
| --- | --- |
| `~/.claude/settings.json` | Claude Code settings (edited) |
| `~/.claude.json` | Claude Code config (read only; changed via `claude mcp`) |
| `~/.config/claude-switch/glm-env.json` | Saved GLM env block, including your key |
| `~/.config/claude-switch/mcp-servers.json` | Saved MCP definitions, including keys |
| `~/.config/claude-switch/backups/` | `settings.json` backup before every change (last 10 kept) |

`CLAUDE_CONFIG_DIR` and `XDG_CONFIG_HOME` are respected if set.

## Which key to use

For GLM Coding Plan quota, use a key from **Personal Coding Plan** or **Team Coding Plan**, not a Standard API key. A Standard key bills your account balance. Details: <https://docs.z.ai/devpack/notice/api-key-management>.

## Security

- The key is never stored in the script or passed on its command line.
- The saved profile and MCP files are `chmod 600` inside a `700` folder, but the key is **plaintext**.
- **Keep `~/.config/claude-switch` out of Dropbox, cloud sync, dotfile repos and backups you share.** Never commit these files.
- `settings.json` and `~/.claude.json` also hold the key in plaintext: `chmod 600` both.
- When the script registers an MCP server, the key is briefly visible in the process list, same as Z.ai's own `claude mcp add` command.
- If a key is ever exposed, rotate it in the Z.ai console, then run `claude-switch glm --new-key`.

## Troubleshooting

**`error: jq is required`**: install `jq` (see Requirements).

**Still on GLM (or asked for an API key) after `claude-switch claude`**
- Close all Claude Code windows and start a new session.
- Check for exported variables: `env | grep ANTHROPIC`. `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` or `ANTHROPIC_BASE_URL` set in `~/.bashrc`, `~/.zshrc` or `~/.profile` can override the subscription. The script warns about these.
- Check for a project-level `.claude/settings.json` or `.claude/settings.local.json` with the same env keys. The script only manages the user-level file.
- Run `claude`, then `/login`.

**MCP servers missing after `claude-switch glm`**
- `claude` must be on your `PATH`. If it wasn't, the script skipped MCP. Install Claude Code, then run `claude-switch glm` again.
- Check with `claude mcp list`.

**A Z.ai MCP server shows as failed**
- The key may be wrong or not a Coding Plan key. Run `claude-switch glm --new-key`.
- For the vision server, confirm Node.js 22+ (`node -v`).

**`settings.json is not valid JSON`**: fix the file, or restore the latest copy from `~/.config/claude-switch/backups/`.

**`settings.json uses a non-Z.ai ANTHROPIC_BASE_URL`**: you have another gateway configured. The script leaves it alone unless you pass `--force`.

## Uninstall

```bash
rm ~/.local/bin/claude-switch
rm -r ~/.config/claude-switch     # also deletes the saved key and backups
```

If you want to stay in a mode, run `claude-switch claude` or `claude-switch glm` first.
