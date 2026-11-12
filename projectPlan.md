# Private customization plan

Status: planning only. No toggle or transfer-behavior changes have been implemented.

Updated: October 28, 2026. Implementation references checked against upstream commit `637ec14b` after syncing the fork.

## Purpose and scope

This is Jeol's private version of RClone Manager. The application will not be published. Keep the existing license and upstream attribution. Jeol is reviewing further changes; record only changes actually requested or discussed, and distinguish proposals from decisions.

Private use is the intended scope, not the current GitHub visibility: as checked on October 28, 2026, `jeolmangx/rclone-manager` remains a public fork of `Zarestia-Dev/rclone-manager`.

The currently requested feature is a toggle to switch folder-transfer behavior back and forth. Implementation remains separate from this documentation work.

## Folder versus contents toggle

### Problem

When selecting a source folder named `Audiobooks` and choosing an existing destination named `Audiobooks`, the current file-manager transfer code appends the source folder name. The resulting path is `Audiobooks/Audiobooks`.

In `transfer()` the backend constructs `dst_file` by appending `item.name` to `dst_path`, then uses that resolved path for the directory's `sync/copy` request. This is destination construction in Manager. Changing a trailing slash is not a dependable way to select transfer semantics.

### Requested outcome

Provide an explicit toggle so the user can choose either behavior for a folder copy:

| Source | Selected destination | Copy folder contents | Resolved directory destination |
| --- | --- | --- | --- |
| `remote:Audiobooks` | `/Media/Audiobooks` | Off | `/Media/Audiobooks/Audiobooks` |
| `remote:Audiobooks` | `/Media/Audiobooks` | On | `/Media/Audiobooks` |
| `remote:Audiobooks` | `/Media` | Off | `/Media/Audiobooks` |
| `remote:Audiobooks` | `/Media` | On | `/Media` |

Contents-copy preserves the hierarchy below the selected source folder. It does not flatten the book subfolders or delete unrelated destination files.

### Proposed UI and defaults

These are the planning defaults discussed so far; adjust them if Jeol specifies a different preference.

- Label: **Copy folder contents**.
- Place the toggle alongside the destination controls for the relevant folder-copy action.
- Off preserves the existing behavior. On uses the selected destination as the final directory destination.
- Show the resolved destination before starting; changing the toggle updates the preview immediately.
- Apply the choice to the current transfer rather than changing all transfers globally.
- Save the choice with a saved task that exposes this behavior. Existing tasks or command payloads without the option retain the current behavior.
- Initially offer contents-copy for one selected directory. Multiple-directory contents merging remains a design decision, not an implicit behavior.
- Keep ordinary file copies and existing copy/paste behavior intact unless the user explicitly selects the new folder-copy mode.

Do not infer the mode from matching folder names, trailing slashes, or whether the destination already exists. Repeated folder names can be intentional.

### Implementation map

1. [Backend transfer command](backend/core/src/rclone/commands/filesystem.rs): add an optional, backward-compatible contents-copy argument. For a selected directory in contents mode, use `dst_path` directly; otherwise retain the existing name-appending behavior. Preserve file handling, remote-root handling, request options, and nested source contents. The current directory-copy request explicitly sets `createEmptySrcDirs: true`.
2. [Frontend command service](src/app/services/remote/remote-file-operations.service.ts): pass the explicit choice through `transferItems()` to the backend. A proposed wire field is `copyContents`; the Rust argument can follow the repository's snake_case convention.
3. [File-manager operation service](src/app/services/ui/nautilus-file-operations.service.ts): carry the choice from the action UI, keep resolved paths consistent, and handle its undo/redo implications.
4. [File-manager component](src/app/file-browser/nautilus/nautilus.component.ts): inspect the actual initiating copy/download flow and its controls before placing the toggle. The exact component placement has not been selected.
5. [Translations](resources/i18n): add user-facing labels through the existing translation system when implementation begins.
6. [Shared command catalog](backend/core/src/app/commands.rs): extend the `transfer` argument declaration alongside the backend signature. The [shared dispatcher](backend/core/src/app/dispatch.rs) deserializes camelCase payload fields; keep an omitted option compatible with old callers. Inspect saved-task persistence before extending it. Keep desktop and headless payload behavior aligned; cover mobile if this action is exposed there. The [server command adapter](backend/server/src/web/commands.rs) calls this shared dispatcher.

Use the same destination-resolution semantics for execution, preview, task replay, and job details. Do not add a second source of truth in the UI.

The current `transfer()` job metadata records the selected destination (`dst_remote` plus `dst_path`), while each request uses its resolved destination including the source name. Inspect how job details consume this metadata before deciding how to represent the resolved destination and mode, especially for multiple items.

### Current repository layout

Upstream replaced the former `src-tauri` source tree with a [backend Cargo workspace](backend/Cargo.toml): shared logic in [core](backend/core), the Tauri/mobile host in [desktop](backend/desktop), and the headless HTTP host in [server](backend/server). Use the current workspace validation commands in [AGENTS.md](AGENTS.md). Old local `src-tauri` build artifacts do not identify the active source location.

### Undo and redo

The current file-manager service records `joinPath(dstPath, item.entry.Name)` as the copied destination. Copy undo subsequently deletes the recorded item, including a directory through the existing deletion path.

For contents-copy, the destination can be a pre-existing folder containing unrelated files. Recording that directory and deleting it on undo would be unsafe.

Proposed first implementation: do not register contents-copy in the existing folder-delete undo stack, and explain the limitation in the action UI. Any later reversible implementation needs to track newly created files and restore overwritten files. Redo must retain the chosen mode and resolved destinations. Simply changing the recorded path is insufficient.

### Acceptance checks for implementation

- Toggle off: a selected `Audiobooks` folder copied into `/Media` appears at `/Media/Audiobooks`.
- Toggle on: the same source copied into `/Media/Audiobooks` places its files directly there, without another `Audiobooks` layer.
- Switching the toggle updates the preview and the submitted directory destination consistently.
- Source subfolders, including empty directories, remain represented correctly.
- Existing unrelated destination files remain untouched; normal rclone copy collision/overwrite options still apply.
- Missing destinations are handled correctly and remote-root destinations remain valid.
- Names containing spaces, Unicode, and repeated path components remain literal; paths are not deduplicated by name.
- Single-file copying keeps its existing destination-filename behavior.
- Old command payloads and saved tasks default to folder-copy semantics; saved tasks with the option retain it on replay.
- Undo never removes a pre-existing destination directory or unrelated contents. The proposed first version excludes contents-copy from the existing undo stack.
- Multiple selection cannot silently turn into an unplanned contents merge.
- Relevant desktop/headless bridge behavior and platform paths are covered by regression tests.

When application work begins, add focused frontend/backend tests and run the relevant checks listed in [AGENTS.md](AGENTS.md). Documentation-only edits do not require building the application.

### Decisions still open

- The exact screen/control placement for the copy/download action Jeol uses.
- Whether to extend the toggle to move operations; the initial plan covers copying only.
- Whether and how multiple source folders may merge their contents.

## Other changes

No other application changes are recorded yet. Add them here as Jeol describes them; the earlier GUI comparison does not imply a request to implement every borrowed pattern.
