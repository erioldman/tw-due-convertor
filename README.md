# tw-hooks

A collection of Taskwarrior hooks I use. Each hook lives in its own directory with its `on-add` script and the `on-modify` symlink that makes it fire on both add and modify.

| Hook        | What it does |
|-------------|--------------|
| [deadline](#deadline)   | Turns date-only due dates into true end-of-day deadlines |
| [inbox-wait](#inbox-wait) | Tasks gaining `+inbox` start waiting until triaged |

## Requirements

- `python3` on PATH.
- The hook directory varies by Taskwarrior version — find yours with `task diagnostics` under **Hooks → Location**. Taskwarrior 3.5 uses `~/.config/task/hooks`; Taskwarrior 2.x uses `~/.task/hooks`.

Verify your hooks are picked up:

```
$ task diagnostics | grep -A5 Hooks
Hooks
     System: Enabled
   Location: /home/you/.config/task/hooks
     Active: on-add.deadline     (executable)
             on-modify.deadline  (executable) (symlink)
             ...
```

## deadline

`task add Pay rent due:tomorrow` normally stores `due` as tomorrow **00:00:00** — the moment the day starts, the task already shows as overdue. This hook rewrites any midnight `due` to **23:59:59** the same day, so the task stays "due" until the day actually ends.

### Example

```
$ task add Pay rent due:tomorrow
$ task 1 export | jq -r .due
20261123T205959Z        # = 2026-11-23 23:59:59 local
```

### Install

```sh
git clone https://github.com/erioldman/tw-hooks.git
cd tw-hooks
install -m 755 deadline/on-add.deadline ~/.config/task/hooks/
ln -s on-add.deadline ~/.config/task/hooks/on-modify.deadline
```

Or without git:

```sh
mkdir -p ~/.config/task/hooks
curl -fsSL https://raw.githubusercontent.com/erioldman/tw-hooks/main/deadline/on-add.deadline \
  -o ~/.config/task/hooks/on-add.deadline
chmod +x ~/.config/task/hooks/on-add.deadline
ln -s on-add.deadline ~/.config/task/hooks/on-modify.deadline
```

(The repo also ships `deadline/on-modify.deadline` as a symlink, so `cp -a deadline/. ~/.config/task/hooks/` works too.)

### What it does

Fires on `on-add` and `on-modify`. For every task whose `due` lands exactly on **00:00:00 local time**, it rewrites it to 23:59:59 the same day. Explicit times are left untouched:

| input                     | stored due              |
|---------------------------|-------------------------|
| `due:tomorrow`            | tomorrow 23:59:59       |
| `due:2026-12-25`          | 2026-12-25 23:59:59     |
| `due:2026-12-25T09:00:00` | unchanged               |

Dates are compared in your local timezone (DST-safe — it is not a naive +86399 seconds) and written back in the same format Taskwarrior uses.

### How it works

Taskwarrior pipes task JSON to every executable hook whose name starts with `on-add` / `on-modify`; the last line the hook prints is the task that gets saved. This script reads stdin, rewrites `due` when it falls on local midnight, and echoes the JSON back. `on-modify.deadline` is a symlink to the same script: for modifications Taskwarrior passes two JSON lines (original, modified) and only the last line matters, so one script serves both hooks.

Handles both Taskwarrior 3.x (RFC 5545 UTC strings, e.g. `20261224T210000Z`) and 2.x (epoch integers).

Note: the hook only affects tasks added or modified after installation — existing midnight dues are not rewritten retroactively.

### Removing

```sh
rm ~/.config/task/hooks/on-add.deadline ~/.config/task/hooks/on-modify.deadline
```

## inbox-wait

Moving a task into the inbox usually means "I'll look at this later" — but an inbox task is still pending, so it keeps showing up in `next` and urgency reports. This hook makes `+inbox` tasks wait automatically: a task gaining `+inbox` gets `wait:later` (unless a wait date is already set), and a task losing `+inbox` gets its `wait` cleared again. Pair it with a `+inbox` context or report to triage only what has come out of waiting.

### Example

```
$ task add Check mail +inbox
$ task 1 export | jq -r .wait
later
$ task 1 modify -inbox
$ task 1 export | jq -r .wait
(empty)
```

### Install

```sh
git clone https://github.com/erioldman/tw-hooks.git
cd tw-hooks
install -m 755 inbox-wait/on-add.inbox-wait ~/.config/task/hooks/
ln -s on-add.inbox-wait ~/.config/task/hooks/on-modify.inbox-wait
```

Or without git:

```sh
mkdir -p ~/.config/task/hooks
curl -fsSL https://raw.githubusercontent.com/erioldman/tw-hooks/main/inbox-wait/on-add.inbox-wait \
  -o ~/.config/task/hooks/on-add.inbox-wait
chmod +x ~/.config/task/hooks/on-add.inbox-wait
ln -s on-add.inbox-wait ~/.config/task/hooks/on-modify.inbox-wait
```

(The repo also ships `inbox-wait/on-modify.inbox-wait` as a symlink, so `cp -a inbox-wait/. ~/.config/task/hooks/` works too.)

### What it does

Fires on `on-add` and `on-modify`:

| change              | result                                  |
|---------------------|-----------------------------------------|
| gains `+inbox`      | `wait:later` set — unless `wait` is already set |
| loses `+inbox`      | `wait` removed                          |
| no `+inbox` change  | untouched                               |

An explicit wait set together with `+inbox` is respected, so `task add Read X +inbox wait:2026-12-01` stays waiting until December 1st, not "later".

### How it works

For `on-add`, Taskwarrior passes one JSON line and the script sees a task with no "original" — gaining `+inbox` from nothing. For `on-modify`, it receives two lines (original, modified) and compares the `tags` of both: it only touches `wait` when `+inbox` presence actually changes between the two lines, so re-saving an inbox task never rewrites an existing wait.

Note: removing the hook does not clear `wait:later` from tasks already tagged — finish triaging first, or clear waits manually.

### Removing

```sh
rm ~/.config/task/hooks/on-add.inbox-wait ~/.config/task/hooks/on-modify.inbox-wait
```
