---
description: Migrate dev setup, apps, and folders from an old Mac on the same network
argument-hint: "<user@old-mac-host> [items: tools, app names, ~/folders…]"
---
You are migrating my setup from an old Mac to this Mac over the local network.

Old Mac: ${1:-ASK ME for user@host}
Items to migrate: ${@:2}

If no items are listed above, scan the old Mac, list candidates (apps in /Applications, dotfiles, CLIs, big folders with sizes), and ask me to pick.

Items can be: CLI tools (node, aws, gh, git, git-ai, go …), app names ("Hidden Bar", "Executor" …), agent tools (pi, herdr, claude …), or folders (~/work, ~/Downloads …).

## Rules
- Pull from the old Mac to this Mac with rsync over SSH. Never delete or change anything on the old Mac. The one exception: quit an app there before you copy its data, and tell me first.
- Before you replace anything on this Mac, back it up to ~/migration-backup-<timestamp>/ and report the path.
- Never print secrets (tokens, keys, passwords). Mask them in output.
- Steps that need a password, a browser, or sudo: give me the exact command, then wait for me.
- Verify each step (versions, `list` commands, integrity checks) and report the result, not assumptions.
- Remote paths that contain spaces must be escaped for the remote shell: `"host:Library/Application\ Support/X/"`.
- End with a table: item → status → what I still need to do.

## 1. Connect
1. Resolve and ping the host. If port 22 is refused, tell me to turn on System Settings → General → Sharing → Remote Login on the old Mac.
2. If ~/.ssh/id_ed25519 does not exist, create it (`ssh-keygen -t ed25519 -N ""`).
3. Test `ssh -o BatchMode=yes`. If key auth fails, tell me to run `ssh-copy-id <host>` on THIS Mac. If the password fails there, give me a one-line command to paste the public key into ~/.ssh/authorized_keys on the old Mac.
4. Check the architecture on both Macs (`uname -m`). Intel → Apple Silicon means native binaries must be rebuilt or reinstalled, not copied.

## 2. Inventory
For each item, find on the old Mac:
- the install location and how it was installed (brew, official installer, .app, version manager)
- config and data paths: ~/.config/<x>, ~/.<x>, ~/.local/state/<x>, ~/Library/Application Support/<x>, ~/Library/Preferences/<bundle-id>.plist, ~/Library/Containers/<bundle-id>
- size (`du -sh`) and whether it runs now

Compare the total with the free disk space on this Mac. Show me the plan and wait for "go".

## 3. Base toolchain (install only what is needed or asked)
- Xcode Command Line Tools: `xcode-select -p`. If missing, `xcode-select --install` (I click through).
- Homebrew: it needs sudo, so I run the official install.sh. Then add `eval "$(/opt/homebrew/bin/brew shellenv)"` to ~/.zprofile and verify `brew --version`.
- Node: match the old Mac's version and version manager (nvm, fnm, volta, or brew). Reinstall global npm packages from the old `npm ls -g --depth=0`.
- Other CLIs: prefer `brew install`, and match what the old Mac used.

## 4. Per-tool recipes
- **git**: copy ~/.gitconfig and ~/.config/git/. Rewrite absolute paths in it (for example /usr/local/bin/gh → /opt/homebrew/bin/gh).
- **gh**: `brew install gh`. Run `gh auth login -h github.com -p https --web` in a detached `screen`, show me the one-time code, and open https://github.com/login/device. Verify with `gh auth status` and `git ls-remote` on a repo.
- **AWS CLI**: install with brew. Without brew, use the official .pkg installed to the user home folder. Copy ~/.aws but exclude cli/cache and sso/cache. chmod 700 the folder and 600 the files. Run `aws configure list-profiles`. For SSO profiles, I run `aws sso login --sso-session <name>`.
- **git-ai**: reinstall with its official installer (`curl -sSL https://usegitai.com/install.sh | bash`). Do not copy the binary. Verify `git-ai --version` and the agent hooks.
- **pi**: copy ~/.pi (settings, extensions, skills, prompts, sessions). Ask whether to keep this Mac's ~/.pi/agent/auth.json. Then run `pi mcp list`. For tokens in mcp.json that change on restart, use `!command` headers that read the current token. For OAuth servers, run `pi mcp login <server>` detached and have me approve in the browser.
- **~/.agents** (shared agent skills): copy it if present.
- **herdr**:
  - Copy ~/.config/herdr (exclude *.sock*, *.log) and ~/.local/state/herdr.
  - Copy ~/.herdr (worktrees) only after their main repos are copied.
  - Check `herdr plugin list` and the architecture of plugin binaries. Rebuild plugins that do not match.
  - To restore sessions: back up session.json → `herdr server stop` (it overwrites session.json!) → move session-snapshots/* aside → restore session.json → I launch `herdr`. Check that the workspace folders exist.
  - Do not stop the server if you run inside herdr (HERDR_ENV=1).
- **GUI apps (.app)**:
  - Find the bundle id (`defaults read <app>/Contents/Info CFBundleIdentifier`).
  - If the app is signed and its architecture matches (`lipo -archs`), rsync the .app and verify it with `codesign --verify --deep --strict`. Otherwise use `brew install --cask`.
  - Quit the app on BOTH Macs. Then copy Application Support (exclude Singleton*, Cache, Code Cache, GPUCache, Dawn*Cache, Crashpad), the Preferences plist, and Containers prefs.
  - Run `killall cfprefsd`, relaunch the app, and check it.
  - Warn me that Keychain logins do not transfer.
- **Apps with a background daemon** (for example Executor): stop the daemon right before you copy its database, because it can restart by itself. Check the database with `sqlite3 <db> "PRAGMA integrity_check"`. Exclude pid, lock, and socket files.

## 5. Large folders
Run the copy in a detached `screen` session, because tool timeouts kill background jobs:

    rsync -ah --partial --stats -e "ssh -o BatchMode=yes" <host>:<dir>/ ~/<dir>/

- Put it in a script I can run again (~/migration-rsync.sh) that logs to ~/migration-rsync.log and prints ALL_DONE at the end.
- Make sure only ONE rsync writes to each target. Check `pgrep -fl rsync` before you start another.
- Report progress as size and MB/min, with an ETA.
- Tell me to keep the old Mac awake (`caffeinate -d`).
- Git worktrees stored outside the copied folders (other tools' worktree dirs): list them with their sizes and ask before you copy them.

## 6. Finish
Give me:
- a summary table
- the backup location
- the manual steps that remain (logins, `/reload`, app restarts)
- what was not copied, and why
