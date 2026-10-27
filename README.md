# claude-sessions

A full-screen terminal dashboard for your running Claude Code sessions (macOS), also served
as a web page. One tile per session with status, context usage, model, tokens, working directory,
git branch, current task, the session's subagents and workflows, plus a usage-limit bar, a
projects bar that opens repos in VS Code, and a chat with a small Claude that acts for you.

## Install

```sh
./install.sh        # symlinks into ~/.claude/scripts and adds the `cs` alias
```

Requires `python3` (stdlib only). Run `cs`.

## Usage

| Command        | What it does                              |
|----------------|-------------------------------------------|
| `cs`           | live full-screen dashboard (+ web view)   |
| `cs -o`        | print once and exit                       |
| `cs -a`        | also show sessions ended in the last 24h  |
| `cs -t`        | classic table layout                      |
| `cs -j`        | JSON output                               |
| `cs -s [PORT]` | web dashboard only (default port 7421)    |
| `cs -w`        | open the web dashboard instead of the TUI |

Keys: arrows select, Enter or click focuses the session's app, `/` opens the chat,
`p` searches projects, `a` toggles ended sessions, `v` switches to [transcripts](#transcripts), `r` refreshes,
`w` opens the web view, `q`/Esc quits.

`cs` also serves the web dashboard in the background and shows an `open web view ↗` link on the bottom
line (click it or press `w`). It uses the same port as `cs -s` (`7421` or `$CS_PORT`); if a
`cs -s` or another `cs` is already serving there, it links to that one instead, and if something
else holds the port it picks a free one. The server stops when `cs` quits.

Focus works for VS Code (window of the session's folder), Terminal.app and iTerm2
(exact tab by tty). Other terminals (e.g. Warp) are only brought to the front.

## Web dashboard

`cs -s` serves the same dashboard as a web page: tiles, agents, context bars, usage limits
bar, projects bar and the chat, refreshed every 2s. It listens on `127.0.0.1:7421` (or a free
port if that one is taken); `cs -s 8080` or `CS_PORT=8080 cs -s` uses a fixed port and fails if
it is busy. The page lives in `web/index.html` and is re-read on every load.

It only accepts requests addressed to `127.0.0.1`/`localhost` that carry a token generated at
startup, so other websites can't drive the chat (which runs with permissions bypassed).

`cs -w` does the same and opens the page in your browser, without the terminal dashboard. If a
`cs` or `cs -s` is already serving on that port, it just opens that one and exits.

`cs -w`, `w` and the `open web view ↗` link open the installed app instead of a browser tab when
there is one for the same port (Chrome, Edge or Brave apps in `~/Applications`).

Keys: arrows select, Enter or click focuses the session's app, `/` opens the chat (Esc closes),
`p` searches projects, `a` toggles ended sessions, `t` switches between tiles and the table layout, `r` refreshes,
`1`, `2` and `3` switch between sessions, [tasks](#tasks) and [transcripts](#transcripts).

It can be installed as an app (its own window and Dock icon): press `i`, or use the install
button in Chrome's address bar; in Safari use File › Add to Dock. The app needs `cs`, `cs -s` or
`cs -w` running on the same port; while the server is down it shows the last page as offline and
reloads itself when the server is back.

### Themes

Pick a theme from the select in the top-right corner. Themes are listed in `.settings.json`,
which also stores the selected one (the select writes it back; editing the file by hand works too,
it is read on every request):

```json
{
  "projects": "~/Projects",
  "theme": "terminal",
  "themes": [
    { "id": "terminal", "title": "Terminal", "file": "Terminal-theme.css" },
    { "id": "claude", "title": "Claude", "file": "Claude-theme.css" }
  ]
}
```

Each theme is one complete stylesheet in `web/themes/`. To add one, copy a theme file, restyle
it, and add an entry to `themes`. The page sets some colours inline, so a theme must define the
tokens listed at the top of `Terminal-theme.css`.

## Projects

The line above the chat lists the git repos you used most recently with Claude (from
`~/.claude/projects`); click one to open it in VS Code. `p` searches all repos in the projects
folder: type to filter, arrows select, Enter opens, Esc closes. In the web view the line is a row
of chips with a search box; the folder at its right end shows the projects folder, click it to
change it.

The projects folder is `projects` in `.settings.json` (default `~/Projects`, colon-separate several).
Repos are found up to three levels deep. `CS_REPO_ROOTS`, if set, overrides it. Projects open
with `open -a "Visual Studio Code"`, so VS Code must be installed.

## Tasks

The web view's `tasks` tab (or `2`) tracks work per project, grouped by client and repo. The client is
the first folder under the projects folder (`~/Projects/ariston/promozioni.ariston.com` belongs to
`ariston`), so a task's project is either `client/repo` or just the client when the repo isn't known yet.

Each task has a kind (`fix` changes code, `check` investigates and answers), a priority, a status and a
history of its changes and notes; tasks can have subtasks. Statuses: new, queued, analyzing, to approve,
implementing, to review, needs info, done, rejected, failed. To approve, to review, needs info and failed
count as needing you: they put the client first and show as `◆` on the tab.

Keys: `n` new task, arrows select, `e` edits the selected one, Esc closes, `a` toggles between open tasks
and all of them, `s` scans the inbox, `m` shows the inbox log. The detail panel sets status, priority and kind, adds subtasks and notes, opens the repo
in VS Code and deletes the task (with its subtasks).

Tasks live in `~/.claude/state/cs-tasks.db` (SQLite, or `$CS_TASKS_DB`).

### Working a task with Claude

`analyze` in a task's detail opens a new terminal tab or window running an interactive `claude` session in the
task's repo, in plan mode, with the task (and its email, marked as third-party data) as the first prompt.
It shows up in the sessions view like any other session, its tile tagged `task #N`, and `focus session`
brings its tab forward. The task follows the session by reading its transcript:

| Status       | When                                                                                     |
|--------------|------------------------------------------------------------------------------------------|
| analyzing    | the session is investigating, read-only                                                  |
| to approve   | Claude posted its plan (findings, for a check): approve or comment in its tab            |
| implementing | you approved: it creates a worktree on `task/<id>-<slug>`, implements, tests and commits |
| to review    | it ended its report with `[task #N ready for review]`                                    |
| failed       | the session ended before posting a plan                                                  |

Rejecting or commenting on the plan sends the task back to analyzing until the next plan. The detail shows
the plan, the final report, and the branch's worktree, commits and diff stat; `open in VS Code` opens the
worktree once it exists. Nothing is pushed, merged or deployed and no email is sent: you review the branch
and merge it yourself, then mark the task done. `resume session` reopens an ended session in the chosen terminal;
`analyze again` starts a new one. The first `analyze` asks macOS to let the dashboard control Terminal.

`sessions in` in the Tasks bar picks the terminal for these sessions, from the ones installed (Terminal,
iTerm, Warp, Ghostty, kitty, Alacritty, WezTerm); it is saved as `terminal` in `.settings.json` and
defaults to Terminal. Warp gets the session's script as a `.command` file to run; only Terminal and
iTerm can select the exact tab on `focus session`, the others are brought to the front.

### Tasks from email

The inbox scan reads your Gmail inbox through the Gmail connector of your Claude account and turns
emails that ask for work on a project in the projects folder into tasks (marked `✉`). It runs every 15
minutes while `cs`, `cs -s` or `cs -w` serves the web view, and on demand with `scan inbox` (or `s`).
The `auto every … min` controls in the tasks tab switch the timer off or change it; they are saved as
`inbox` in `.settings.json`:

```json
"inbox": { "auto": true, "every": 15, "days": 14 }
```

Each scan is a short headless Claude run (`claude -p`, `$CS_INBOX_MODEL`, default `haiku`) that sees only
`in:inbox` threads from people other than you with a message newer than the last good scan, and never
older than `days` (two weeks): the first scan, or one after `cs` was off for a while, covers the last
two weeks only. It classifies them by sender and subject, reads a thread only to summarise a task, and picks the
project as `client/repo`, or just the client when the repo isn't clear. CI runs, notifications, orders,
newsletters, invites and mail about anything outside the projects folder are skipped. A new reply in a
thread that is already a task is added to that task's history instead of creating a new one.

The run can only search and read mail: its built-in tools are off, every Gmail tool that writes (send,
reply, forward, drafts, labels, trash, spam) is denied, any MCP tool your settings pre-approve is denied,
and anything else is refused without asking. It never changes your mailbox. A scan with no new mail takes
seconds and costs a fraction of a cent; the first one, over two weeks of mail, costs around $0.20 to $0.40.

`inbox log` (or `m`) lists every thread the scans looked at, what they decided and why, with a link to
Gmail. The scan learns from your corrections, which it reads as examples on every run:

- `make task` on a skipped thread creates the task and tells later scans that mail like it is a task;
- `not a task` on an email task rejects it and tells later scans the opposite;
- changing an email task's project tells later scans where mail like it belongs.

## Transcripts

The transcripts view (`v` in the TUI, `3` or the `transcripts` tab on the web) shows one chat box per
session: its conversation read from the transcript (your messages, Claude's replies and its tool calls,
without subagents) with an input line at the bottom. Select a box and press Enter (or click it) to type;
Enter sends. Where the message goes depends on the session:

| Session                      | Sending                                                                        |
|------------------------------|--------------------------------------------------------------------------------|
| run by cs                    | straight to it, see below                                                      |
| ended                        | resumes it as a session run by cs, with your message                           |
| live in Terminal or iTerm2   | typed into its tab (found by tty), as if you typed it there                    |
| live in Warp, VS Code, other | read-only: those apps can't be typed into from outside, `f`/`focus` brings it up |

### Sessions run by cs

`cs` can run sessions itself, with no terminal: `claude -p` in stream-json mode, with its permission
prompts sent to the dashboard (`--permission-prompt-tool stdio`). Start one with `new session in` above
the boxes on the web (folder, permission mode, optional first message), or `n` in the TUI (pick the
folder in the projects search). Task sessions can run this way too: pick `dashboard (cs)` in the Tasks
bar's `sessions in`, and Analyze starts the session in plan mode inside cs.

When Claude asks to use a tool, or posts a plan, the box shows it with `allow`/`approve` and `deny`
(`y` and `d` in the TUI). Typing a reply instead denies it with your message, which is how you comment
on a plan. `stop` (`x`) interrupts the current turn; on an idle session it becomes `close`, which ends
the process so you can resume the session anywhere.

These sessions are child processes of the `cs` that started them: quitting that `cs` ends them (they show
as ended and resume from their box). Only that `cs` can drive them, so if the TUI links to a web view
served by another `cs`, start them from the one you use.

Typed messages to Terminal and iTerm2 are sent as one line (newlines become spaces, since a newline would
submit early). Sending to them asks macOS once to let the dashboard control that app.

## Chat

Press `/` (or click the chat box) to talk to a small Claude of its own, separate from
your sessions: "open claude-sessions in VS Code", "show ariston in Finder". Enter sends,
Esc closes, `/clear` starts a new chat. It knows the git repos in the projects folder
and your live sessions, and runs headless (`claude -p`) with every tool and MCP server
and permissions bypassed, so it acts without asking.

| Variable          | Default                                         |
|-------------------|-------------------------------------------------|
| `CS_CHAT_MODEL`   | `haiku`                                         |
| `CS_REPO_ROOTS`   | overrides the projects folder (colon-separated) |
| `CS_CHAT_ROWS`    | `8` chat lines shown                            |

## Other settings

| Variable            | Default                                                    |
|---------------------|------------------------------------------------------------|
| `CS_PORT`           | `7421` web dashboard port (`cs`, `cs -s`, `cs -w`)         |
| `CS_TASKS_DB`       | `~/.claude/state/cs-tasks.db` tasks database               |
| `CS_INBOX_MODEL`    | `haiku` model for the inbox scan                           |
| `CLAUDE_CTX_WINDOW` | context window for the bars (`200k`, or `1M` past 200k)    |

## Agents

Each tile lists the subagents (Agent tool) and workflows the session started. Running ones
show their type, description, current tool (or the workflow's phase and agents done/started)
and elapsed time. Finished ones stay listed for 60s with ✓ (done), ✗ (failed) or ■ (stopped),
then only count in the `done` total. The header shows how many agents are running across all
sessions.

| Variable          | Default                                         |
|-------------------|-------------------------------------------------|
| `CS_AGENT_ROWS`   | `4` agent lines per tile (then `+N more`)       |

## Colours

- orange logo: always; rainbow when the session is in ultracode
- orange status: running, grey: waiting / ended

## Usage limits bar

Claude Code only passes rate limits to the status line. Add
`extras/statusline-snippet.sh` to your statusLine script so it writes
`~/.claude/state/rate-limits.json`, which the dashboard reads. The snippet needs `jq`.

## Open at login (Warp)

Edit `YOUR_USER` in `extras/warp-launch-configuration.yaml`, then copy it to `~/.warp/launch_configurations/`
and `extras/com.user.claude-sessions.plist` to `~/Library/LaunchAgents/`, then
`launchctl load` it.

## Data sources

- live sessions: `~/.claude/sessions/*.json`
- transcripts: `~/.claude/projects/*/<sessionId>.jsonl`
- agents: `~/.claude/projects/*/<sessionId>/subagents/` (agent transcripts, workflow journals)
- limits: `~/.claude/state/rate-limits.json`
- recent projects: dates of the history folders in `~/.claude/projects/`
- settings: `.settings.json` next to the script
- tasks: `~/.claude/state/cs-tasks.db`
