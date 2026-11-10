# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file demo/training Kanban board for a fictitious bank's IT PMO. The whole app is `index.html` (markup, `<style>`, `<script>`). There is no build, lint, or test tooling; run it by opening the file in a browser (double-click, `file://`, no server).

## Hard constraints (from the original brief)

- Vanilla HTML/CSS/JS only: no frameworks, bundler, npm, CDN scripts, web fonts, or image files. Icons are Unicode glyphs/inline SVG.
- No persistence: no localStorage/sessionStorage/IndexedDB/cookies. Board state resets to seed data on refresh, and the UI says so.
- No `!important` in CSS; colours and spacing come from `:root` custom properties.
- Only network call is FormSubmit's AJAX endpoint (`FORMSUBMIT_ENDPOINT`). Never send the user's email anywhere else.
- Do not use real UOB branding; the wordmark is neutral text "IT PMO". (Task IDs still use the specified `UOB-ITPM-####` format.)

## Architecture

State flows one way: mutate `state` -> `renderBoard()` rebuilds the board DOM.

- `state = { tasks, filters, ui }` is the single source of truth. `ui` holds transient flags (`moveOpenId`, `deleteConfirmId`, `focusSel`) so the Move menu and inline "Delete? Yes/No" survive re-renders; `focusSel` is a selector `renderBoard()` focuses after replacing the DOM (keyboard users would otherwise lose focus).
- Card HTML is produced only by `renderCard()` via `renderBoard()` (innerHTML strings). Every user-supplied value must go through `escapeHtml()`; toasts use `textContent`.
- Board interaction is event-delegated on `#board` in `initBoardEvents()` (button `data-action` clicks plus native drag/drop), so listeners survive the full re-render. Drag/drop and the keyboard "Move ▸" menu both end in `moveTask()`.
- Add Task flow (`handleSubmit`): `validateForm` -> `addTask` (optimistic render) -> `notifyNewTask` in try/catch. A failed email only shows a warning toast; it must never break the board. The modal stays open with a disabled "Sending…" button until the request settles.
- `notifyNewTask` deliberately throws without calling the network while the endpoint still contains the `YOUR_EMAIL@example.com` placeholder.
- IDs come from `nextTaskNumber` (seeds consume 0001–0008); seed due dates are offsets from today so Overdue badges stay demonstrable.
- Overdue = `dueDate < today` and `status !== "Done"`, comparing local-date `YYYY-MM-DD` strings (`dateISO`).
- Filters apply only to the board columns (`applyFilters`); the header summary strip counts all tasks.
