# gig

One Claude Code agent per ticket, each in its own git worktree, in a tmux window (or a [Herdr](https://herdr.dev) space).

`gig SHOP-1234` finds the ticket's branch (or makes `SHOP-1234-<slug of its summary>` from the right base), makes a worktree next to the project's checkout with the git-ignored files it needs, and starts `claude` there with a first message. `gig close SHOP-1234` takes it all down again: windows, processes, worktree and local branch, never work that would be lost.

The full reference is the docstring at the top of [`bin/gig`](bin/gig).

## Requirements

- Python 3, git, [fzf](https://github.com/junegunn/fzf), tmux, the `claude` CLI
- macOS Keychain (`security`) for the tracker's and code host's tokens; `gh` for GitHub
- pnpm, if a project installs dependencies in new worktrees

## Install

```sh
git clone <this repository> && cd gig
ln -s "$PWD/bin/gig" ~/.local/bin/gig
ln -s "$PWD/bin/roger" ~/.local/bin/roger   # only for Herdr mods that call roger
```

## A client's settings: gig.json

gig reads `gig.json`, the nearest one above the current folder: one per client, in the folder that holds the client's repositories. `gig init`, run in that folder, writes it asking step by step:

- the ticket tracker (Jira, with the token in the Keychain) and `ticket_info`, a command that prints a ticket as JSON (`{"summary", "subtask", "parent"}`)
- the code host (GitLab or GitHub) for merge requests
- `commands`, the folder of the scripts and Claude commands the projects share
- `agent_args`, the flags every `claude` starts with
- `tasks`: the first messages for working on a ticket (`{key}`), reviewing a merge request and answering a review (`{url}`)
- the projects: ticket prefix, base branch, the git-ignored files a new worktree needs (`CLAUDE.md`, `.claude`, …), `pnpm install` or not

Paths in `gig.json` are relative to its folder. Outside a client's folder gig stops, unless `PROJECTS_FILE` names a projects file in the older format.

## A Claude Code plugin per client

When the folder above the client's folder is a local [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) (it has `.claude-plugin/marketplace.json`), `gig init` also makes the client's plugin in `<client>/_claude-plugin`, lists it in the marketplace under the client folder's name, and enables it in each project with local scope (`.claude/settings.local.json`, skipped where git does not ignore that file). The client's skills then show up only in that client's projects, as `/<client>:<skill>`.

## Using it

```
gig                       the session's tickets with their agents; pick one, or type a key, a branch or a link
gig KEY [ACTION]          work (default), review, answer, close, tag, or a /command or /<plugin>:<skill>
gig open KEY [--task TEXT] [-p PROJECT] [-n]
gig close KEY|FOLDER [-f] [-n]
gig links [SESSION]       the session's tickets with links to each ticket and its merge request
gig sidebar --toggle SESSION
gig notify busy|waiting|done|ready|end
gig tag TEXT|--round|--clear
gig init [--yes]
```

`review` and `answer` find the ticket's merge request and open its branch with `tasks.review` or `tasks.answer`. The action picker offers the client plugin's skills and the `.md` files of `commands`.

`gig tag` puts a reminder on the current branch (`answered`, `waiting for the shop`), shown in the ticket list and the sidebar. `--round` makes it `R<n> ✓`, the branch's next review round, so `tasks.review` can end with "run `gig tag --round`". The tags live in `~/.local/state/gig/tags.json` (`$XDG_STATE_HOME`).

## tmux

```tmux
# prefix+T: gig in a popup, from the current pane's folder
bind T display-popup -E -w 80% -h 60% -d "#{pane_current_path}" "~/.local/bin/gig"
# prefix+u: the session's tickets and their links
bind u display-popup -E -w 80% -h 50% "~/.local/bin/gig links"
# prefix+b: the sidebar, the session's windows with each agent's state
bind b run-shell -b "~/.local/bin/gig sidebar --toggle '#{session_id}'"
```

The sidebar follows you from window to window with a few hooks:

```tmux
set-hook -g 'session-window-changed[90]' {
  if -F '#{&&:#{W:#{P:#{?#{==:#{pane_id},#{@gig_sidebar}},x,}}},#{==:#{P:#{?@gig_sidebar_pane,x,}},}}' {
    run-shell -C "join-pane -d -f -h -b -l #{?@gig_sidebar_width,#{@gig_sidebar_width},32} -s '#{@gig_sidebar}' -t '#{window_id}'"
    wait-for -S gig-sidebar
  }
  run-shell -b "~/.local/bin/gig sidebar --follow '#{session_id}'"
}
set-hook -g 'pane-exited[90]' { run-shell -b "~/.local/bin/gig sidebar --follow '#{session_id}'" }
set-hook -g 'window-linked[90]' { wait-for -S gig-sidebar }
set-hook -g 'window-unlinked[90]' { wait-for -S gig-sidebar }
set-hook -g 'window-renamed[90]' { wait-for -S gig-sidebar }
set-hook -g 'alert-bell[90]' { wait-for -S gig-sidebar }
set-hook -g 'client-session-changed[90]' { wait-for -S gig-sidebar }
```

The agent's state goes in the window option `@gig_state`: add `#{?@gig_state, #{@gig_state},}` after `#W` in `window-status-format` and `window-status-current-format` to see it beside each window's name.

## Claude Code hooks

`gig notify` keeps each agent's state (⏳ asks you something, 🔨 works, ✓ your turn) and rings the window's bell when it needs you. In `~/.claude/settings.json`:

```json
"hooks": {
  "SessionStart": [{ "matcher": "startup|resume|clear", "hooks": [{ "type": "command", "command": "~/.local/bin/gig notify ready", "async": true }] }],
  "UserPromptSubmit": [{ "hooks": [{ "type": "command", "command": "~/.local/bin/gig notify busy", "async": true }] }],
  "PostToolUse": [{ "hooks": [{ "type": "command", "command": "~/.local/bin/gig notify busy", "async": true }] }],
  "Notification": [{ "matcher": "permission_prompt|elicitation_dialog|elicitation_url_dialog", "hooks": [{ "type": "command", "command": "~/.local/bin/gig notify waiting", "async": true }] }],
  "Stop": [{ "hooks": [{ "type": "command", "command": "~/.local/bin/gig notify done", "async": true }] }],
  "StopFailure": [{ "hooks": [{ "type": "command", "command": "~/.local/bin/gig notify done", "async": true }] }],
  "SessionEnd": [{ "hooks": [{ "type": "command", "command": "~/.local/bin/gig notify end", "async": true }] }]
}
```

## Herdr

Outside tmux gig opens Herdr spaces. `herdr/roger` is a Herdr mod with a dialog to open and close tickets:

```sh
ln -s "$PWD/herdr/roger" ~/.config/herdr-app/mods/roger
```

and `"cmd-shift-j": "mod.open:roger"` in `~/.config/herdr-app/keys.json`.

## License

GPL-3.0-or-later, see [LICENSE](LICENSE).
