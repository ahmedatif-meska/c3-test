# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

The repo is a single file, `kanban.html` ("Flowlane", a Kanban project-management app). It has no build step, package manager, tests, linter or git history. Everything is vanilla HTML/CSS/JS in one file (~1700 lines): `<style>` (lines ~10–394), markup (~395–642), then one `<script>` (~643–end). The only external dependency is Google Fonts (Inter, Sora).

**Run:** open `kanban.html` in a browser (or serve the directory statically). Verify changes manually in the browser.

## Architecture

The script is organised into numbered sections (see the `/* === N. NAME === */` banners):

- **State:** one global `state` object (`boards[]`, `activeBoardId`, `me`, theme, etc.). `board()` returns the active board; each board holds `columns`, `tasks` (flat array with `colId`, `archived`), and `activity`. Transient UI state (page, search, filters, sort) lives separately in `ui` and is never persisted.
- **Persistence:** `Store` (key `flowlane:v1`) tries `window.storage` first, then `localStorage`, then an in-memory fallback. Saves are debounced 250ms via `save()`.
- **Mutation flow:** `pushUndo()` snapshots `JSON.stringify(state)` (30-deep stack) before a change; `commit(activityText)` logs to the activity feed, saves, and calls `renderAll()`. Rendering is full re-render from state (`renderChrome`, `renderDashboard`, `renderBoard`), not incremental DOM patching, so mutate `state` and then render rather than editing the DOM.
- **Templating:** HTML is built with template strings; always pass user data through `esc()`.
- **Task editing:** the modal edits a `draft` copy of the task (`openTaskModal` → `readDraft` → `saveTaskModal`), so cancelling discards changes.
- **Drag and drop:** native HTML5 DnD on `#board` using a shared `placeholder` element and `afterElement()` to compute drop index. Handles both task and column dragging (`dragTaskId` / `dragColId`).
- **Event handling:** delegated listeners on `#board`, `#tChecklist`, etc., using `data-*` attributes (`data-id`, `data-i`, `data-restore`, `data-purge`). Startup is the async `init()` IIFE at the end of the script.
- **Column semantics:** a column's role (todo/progress/done) is inferred by `kindOf(col)` from its title/position, not stored. Dashboard stats and `isDone()` depend on this.
- **Responsive:** `NARROW` media query (≤860px) relocates the board switcher (`placeSwitcher`). Theme is applied through `data-theme` on `<html>` (`applyTheme`).
- **Import/export:** JSON files shaped `{ type:'flowlane-board', version:1, board }`.

## Project targets

- **Supabase:** project `c3-test`, ref `xxjdwhdoeakmsfzrtqhg` (eu-west-2), URL `https://xxjdwhdoeakmsfzrtqhg.supabase.co`. Use this project for all Supabase MCP calls.
- **GitHub:** `ahmedatif-meska/c3-test` (`origin`, branch `main`).

## Supabase persistence

`Store` (section 2 of the script) syncs state to the `c3-test` project: `load()` rebuilds `state` from the tables via `fromRows()`; `save()` writes localStorage immediately, then `pushDiff()` upserts/deletes only rows that changed since the last sync (`toRows()` flattens state; serialized through `syncQueue`). A `flowlane:v1:dirty` flag makes the next load prefer the local copy if a sync failed. `members` and `labels` tables mirror the `MEMBERS`/`LABELS` constants (not synced from the app). Tables: boards, columns, tasks, task_labels, checklist_items, comments, attachments, activity, app_settings, members, labels — all with RLS and open anon policies (no login). Schema changes go through Supabase migrations, then update `toRows`/`fromRows` together.
