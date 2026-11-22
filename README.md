# tw-due-convertor

A Taskwarrior hook that turns date-only due dates into true deadlines.

`task add Pay rent due:tomorrow` normally stores `due` as tomorrow **00:00:00** — the moment the day starts, the task already shows as overdue. This hook rewrites any midnight `due` to **23:59:59** the same day, so the task stays "due" until the day actually ends.

## Example

```
$ task add Pay rent due:tomorrow
$ task 1 export | jq -r .due
20261123T205959Z        # = 2026-11-23 23:59:59 local
```

## Install

```sh
git clone https://github.com/erioldman/tw-due-convertor.git
cd tw-due-convertor
mkdir -p ~/.config/task/hooks
install -m 755 on-add.deadline ~/.config/task/hooks/
ln -s on-add.deadline ~/.config/task/hooks/on-modify.deadline
```

Or without git:

```sh
mkdir -p ~/.config/task/hooks
curl -fsSL https://raw.githubusercontent.com/erioldman/tw-due-convertor/main/on-add.deadline \
  -o ~/.config/task/hooks/on-add.deadline
chmod +x ~/.config/task/hooks/on-add.deadline
ln -s on-add.deadline ~/.config/task/hooks/on-modify.deadline
```

Verify the hooks are picked up:

```
$ task diagnostics | grep -A5 Hooks
Hooks
     System: Enabled
   Location: /home/you/.config/task/hooks
     Active: on-add.deadline    (executable)
             on-modify.deadline (executable) (symlink)
```

## What it does

Fires on `on-add` and `on-modify`. For every task whose `due` lands exactly on **00:00:00 local time**, it rewrites it to 23:59:59 the same day. Explicit times are left untouched:

| input                     | stored due              |
|---------------------------|-------------------------|
| `due:tomorrow`            | tomorrow 23:59:59       |
| `due:2026-12-25`          | 2026-12-25 23:59:59     |
| `due:2026-12-25T09:00:00` | unchanged               |

Dates are compared in your local timezone (DST-safe — it is not a naive +86399 seconds) and written back in the same format Taskwarrior uses.

## Compatibility

- Handles both Taskwarrior 3.x (RFC 5545 UTC strings, e.g. `20261224T210000Z`) and 2.x (epoch integers).
- The hook directory varies by version — find yours with `task diagnostics` under **Hooks → Location**. Taskwarrior 3.5 uses `~/.config/task/hooks`; Taskwarrior 2.x uses `~/.task/hooks`.
- Requires `python3` on PATH.

## How it works

Taskwarrior pipes task JSON to every executable hook whose name starts with `on-add` / `on-modify`; the last line the hook prints is the task that gets saved. This script reads stdin, rewrites `due` when it falls on local midnight, and echoes the JSON back. `on-modify.deadline` is a symlink to the same script: for modifications Taskwarrior passes two JSON lines (original, modified) and only the last line matters, so one script serves both hooks.

## Removing

```sh
rm ~/.config/task/hooks/on-add.deadline ~/.config/task/hooks/on-modify.deadline
```

Note: the hook only affects tasks added or modified after installation — existing midnight dues are not rewritten retroactively.
