# pi-mac-migrate

A [pi](https://pi.dev) prompt template that moves your dev setup from an old Mac to a new one over the local network. The agent does the work; you only type passwords and approve browser logins.

## Install

```bash
pi install git:github.com/sviataslau/pi-mac-migrate
```

## Use

```
/mac-migrate <user@old-mac-host> [items…]
```

Example:

```
/mac-migrate me@macbook-pro.lan node pi herdr aws gh git git-ai "Hidden Bar" Executor ~/work ~/Downloads
```

Without items, the agent scans the old Mac, lists what it found (with sizes), and asks you what to copy.

Items can be:

- CLI tools: `node`, `aws`, `gh`, `git`, `git-ai`, `go`, …
- Agent tools: `pi`, `herdr`, `claude`, …
- App names from `/Applications`: `"Hidden Bar"`, `Executor`, …
- Folders: `~/work`, `~/Downloads`, …

## What it does

1. Connects over SSH (helps you turn on Remote Login and install a key).
2. Makes an inventory and a plan, then waits for your "go".
3. Installs the base tools: Xcode CLT, Homebrew, Node, and CLIs.
4. Copies the config for each tool, with recipes for git, gh, AWS CLI, git-ai, pi (including MCP servers), herdr (plugins, worktrees, sessions), and GUI apps.
5. Copies large folders with `rsync` in a detached `screen` session that you can resume.
6. Reports a summary table and the manual steps that remain.

## Safety

- It only reads from the old Mac. The one exception is quitting an app there before it copies the app's data, and it asks first.
- It backs up everything it replaces to `~/migration-backup-<timestamp>/`.
- It does not print secrets.
- Keychain items do not transfer. You may have to sign in to some apps again.

## Requirements

- Both Macs are on the same network.
- Remote Login is on for the old Mac.
- You have pi on the new Mac.

## License

MIT
