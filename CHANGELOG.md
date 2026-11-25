# Changelog

All notable changes to this Taskwarrior hook collection will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.1.0] - 2026-11-25

### Added

- Hook collection layout: one directory per hook, each with its `on-add` script and a committed `on-modify` symlink.
- `deadline` hook: rewrites a `due` falling on local midnight (00:00:00) to 23:59:59 the same day, preserving the original format (Taskwarrior 3.x RFC 5545 UTC strings and 2.x epochs), DST-safe. Previously published as the `tw-due-convertor` repo.
- `inbox-wait` hook: a task gaining `+inbox` gets `wait:later` unless a wait date is already set; a task losing `+inbox` gets `wait` cleared. One script, two symlinks (`on-add.inbox-wait`, `on-modify.inbox-wait`).
