# slack-axi

A zero-dependency Slack CLI for agents, acting as you: read channels and threads, search, post, upload and download attachments. One Python 3 file using only the standard library: no Node, no npm, no build step. The token lives in the macOS Keychain.

It follows [AXI](https://axi.md), a set of design rules for agent-first CLIs: compact table output with few default fields, long text truncated unless `--full`, explicit empty results, errors on stderr, and a hint for the next command.

```
$ slack-axi search 'release in:#general' --limit 2
matches[2]{channel,time,user,text,permalink} total=14:
  #general,2026-10-02 17:40,Alex,Release 5.11 is out,https://….slack.com/archives/C…/p…
  #general,2026-09-30 09:12,Sam,Release notes draft ready,https://….slack.com/archives/C…/p…
→ slack-axi thread <permalink>
```

It calls the Slack Web API with a **user token** (`xoxp-…`). A user token can only do what you can do in Slack: it sees your channels and DMs, nothing else.

## Install

```bash
git clone https://github.com/scriptease/slack-axi ~/github/slack-axi
ln -s ~/github/slack-axi/slack-axi /opt/homebrew/bin/slack-axi    # or any directory on your PATH
```

## 1. Create the Slack app (once per workspace)

Slack has no personal access tokens, so every token comes from an app.

1. Open https://api.slack.com/apps → **Create New App** → **From a manifest**.
2. Pick the workspace. Many workspaces let any member create and install an app here. The workspace settings menu (cogwheel) is admin-only, but the apps page is not.
3. Paste [`app_manifest.json`](app_manifest.json) (JSON tab) → **Next** → **Create**.
4. **Install to Workspace** → **Allow**. In workspaces that require app approval, this becomes a request to a workspace admin instead.

The manifest only requests user scopes, no bot:

| Scope | Used by |
|---|---|
| `channels:read` `groups:read` `mpim:read` `im:read` | `channels`, resolving `#name` |
| `channels:history` `groups:history` `mpim:history` `im:history` | `read`, `thread` |
| `im:write` | `@user` (opens the DM) |
| `users:read` | `users`, user names in output |
| `search:read` | `search` |
| `files:read` | `download` |
| `files:write` | `upload` |
| `chat:write` | `post` |

Drop scopes you don't need before creating the app. For example, without `im:history` the app can't read your DMs. Changing scopes later requires reinstalling the app.

## 2. Log in

1. In the app settings, open **OAuth & Permissions** and copy the **User OAuth Token** (`xoxp-…`).
2. Store it in the macOS Keychain:

   ```bash
   slack-axi login            # paste the token at the prompt
   slack-axi whoami           # user, team and granted scopes
   ```

The token is stored as a generic password, service `slack-axi`, account = workspace name.

### Several workspaces

`$SLACK_WORKSPACE` selects the Keychain entry (default `default`):

```bash
SLACK_WORKSPACE=personal slack-axi login
SLACK_WORKSPACE=personal slack-axi channels
```

`$SLACK_TOKEN` overrides the Keychain entirely, e.g. for CI.

## 3. Use

```bash
slack-axi channels                         # your channels and DMs (cached 1 day, --refresh)
slack-axi read '#general' --since 2d       # messages, oldest first
slack-axi thread <permalink>               # a thread from a Slack message link
slack-axi search 'release in:#general'     # Slack search syntax
slack-axi post '#general' 'Hello'          # or: echo text | slack-axi post @user -
slack-axi upload @alex build.zip --comment 'New build'
slack-axi download F0123ABCD --out /tmp    # file ids appear in read/thread output
slack-axi raw conversations.info channel=C0123ABCD
```

`<channel>` is an id, `#name`, `name`, `@user` (DM) or a message permalink. `slack-axi help` lists all commands and flags.

## Other Slack CLIs for agents

Same idea, different trade-offs. Pick the one that fits:

| Project | Runtime | Auth | Writes |
|---|---|---|---|
| **slack-axi** (this) | single Python file, stdlib only | `xoxp`, Keychain | post, upload, download |
| [JarvusInnovations/slack-axi](https://github.com/JarvusInnovations/slack-axi) | Node/bun, `npm install -g slack-axi` | `xoxp`, guided `auth setup` | drafts, posted only after explicit `draft send` |
| [lucas-codes/slack-axi](https://github.com/lucas-codes/slack-axi) | Node, build with bun | `xoxp` via env | read-only, GET allowlist |
| [cyberuni/cyber-slack](https://github.com/cyberuni/cyber-slack) | Node, `npm install -g cyber-slack` | `xoxp` or `xoxb` via env | post, reactions, user groups; also an MCP server |

Note: `npm install slack-axi` installs the JarvusInnovations tool, not this one.

## Notes

- **Free plan:** messages and search only reach back 90 days.
- **Caches:** the `users` and `channels` lists are cached for a day under `~/.cache/slack-axi/<workspace name>/`. If you store another workspace's token under the same name, or use `$SLACK_TOKEN`, run `channels --refresh` and `users --refresh`.
- **Sharing one app:** colleagues can share one app, each with their own token. The app page only shows the token of the person viewing it, and only app collaborators can open it, so add them under **Collaborators** and have each one install the app once.
- **Not yet done:** an OAuth browser login (`login` without copying the token from the app page). `login` currently passes the token to `security` as a command-line argument, so it is briefly visible to other local processes.
