# Untitled Markdown Preview Design

**Date:** 2026-07-27
**Status:** Approved for planning

## Problem

Markdown Preview can render only documents associated with a local file. Users who already have Markdown on the clipboard must create and save a file before they can use the app’s complete renderer.

The renderer itself is not file-bound. `MarkdownHTML.render`, `MarkdownWebView.display`, and `ContentViewController.display` accept an in-memory Markdown string. The file requirement comes from the surrounding document lifecycle:

- `AppDelegate` suppresses untitled documents and presents an open panel when no file is supplied.
- `DocumentWindowController.display` renders only when it receives a file URL.
- Edit Mode is disabled without a file URL.
- The editor save path rejects a missing file URL.
- `MarkdownDocument` explicitly disables AppKit’s save machinery.

## Goals

1. Let users launch the app, paste Markdown into an unsaved document, and view it through the complete existing renderer without first creating a file.
2. Make the same workflow available through File → New / Command-N.
3. Preserve pasted and edited source in memory until the user saves or discards it.
4. Let users optionally save the source as a normal Markdown file.
5. Reuse the existing editor, renderer, outline, inspector, export pipeline, conflict handling, and document windows.

## Non-goals

- A second Markdown parser or rendering engine.
- A dedicated clipboard-only window or automatic clipboard reads.
- A side-by-side source and preview layout.
- Autosaving untitled drafts between launches.
- Changing rendered-output export; PDF, HTML, and PNG remain under Export.

## User Experience

### Launch

Launching Markdown Preview without an incoming file opens a blank document titled **Untitled**. It starts in Edit Mode with the editor focused, ready for typing or Command-V.

Opening the app with a file URL continues to open that file directly. File → Open / Command-O remains the explicit file-opening path.

### New documents

File → New / Command-N creates another untitled document. File → New Tab / Command-T creates an untitled document in the current window’s tab group. Opening a file in a new tab remains available from the project navigator.

### Edit and preview

The existing Edit toolbar control and Command-E switch between:

- Edit Mode, containing the in-memory Markdown source.
- The existing full WKWebView preview, including frontmatter, tables, task lists, syntax highlighting, Mermaid, KaTeX, outline navigation, and other current rendering behavior.

Switching to preview does not save. It captures the editor buffer, renders that buffer, and keeps the unsaved baseline so returning to Edit Mode resumes the same draft.

### Saving

Command-S on an untitled document opens `NSSavePanel` with `Untitled.md` as the suggested filename. The save panel grants sandbox write access. A successful save:

1. Writes the Markdown source using the existing atomic-write-first behavior.
2. Associates the document and window with the selected URL.
3. Updates the title, recent documents, sidebar, inspector, Share/Open actions, and document metadata.
4. Starts file watching and retains existing external-edit conflict handling.
5. Renders again with the saved file’s parent directory as the asset base so relative images and links resolve.

Cancelling the panel leaves the draft untouched. A write failure leaves the document dirty and visible; no source is discarded.

Save As uses the same panel and adoption flow for both untitled and file-backed documents.

### Closing and quitting

An unchanged blank document closes without prompting. A modified untitled document uses the existing Save / Don’t Save / Cancel decision:

- **Save:** opens the save panel and closes only after a successful write.
- **Don’t Save:** discards the in-memory draft and closes.
- **Cancel:** returns to the document unchanged.

Application termination waits for the same resolution, matching file-backed editor sessions.

## Architecture

### `MarkdownDocument`

Continue storing Markdown in memory independently of `fileURL`. Untitled windows are valid documents with an empty initial source. The window controller remains responsible for interactive editor saves, preserving the current external-edit and sandbox behavior rather than introducing a second save path.

The document model must accept source replacement before a URL exists. Once saved, it adopts the URL and clears AppKit’s change state as it does for opened files.

### `AppDelegate`

Replace the no-file open-panel launch behavior with standard untitled-document creation. Keep incoming URL and explicit Open behavior unchanged. Avoid creating both an untitled window and a file window during launch.

### `DocumentWindowController`

Treat `currentFileURL` as optional document metadata, not proof that content exists:

- Always render `currentMarkdown`, using a nil asset base for untitled content.
- Enable and enter Edit Mode whenever `currentMarkdown` exists.
- Start untitled documents in Edit Mode and focus the editor.
- Capture editor drafts and render them even when no file URL exists.
- Route an untitled save through `NSSavePanel`.
- Centralize successful URL adoption so initial Save and Save As update every file-backed subsystem consistently.
- Keep disk-conflict checks for file-backed documents only.

### `MainSplitViewController` and rendering components

No new rendering surface is required. The existing layered preview/editor host already keeps both WebKit views warm and transfers scroll position between them. It will receive a nil source URL and asset base until the document is saved.

### Relative assets

Before saving, relative filesystem references have no meaningful base directory and may not resolve. Absolute and remote references retain their existing behavior. After saving, rerender using the selected file’s parent directory so relative references begin resolving immediately.

## Data Flow

### Paste to preview

1. AppKit creates an untitled `MarkdownDocument`.
2. `DocumentWindowController` displays an empty in-memory source and enters Edit Mode.
3. The user pastes or types; the editor marks the controller dirty.
4. The user invokes Command-E or the Edit toolbar control.
5. The controller fetches the editor buffer, stores it as the draft/current source, and sends it through `MainSplitViewController.display` with nil source and asset URLs.
6. The normal renderer produces the full preview.

### Save

1. The controller fetches the latest editor buffer or uses the retained preview draft.
2. Because no URL exists, it presents `NSSavePanel`.
3. On approval, the controller writes the source and adopts the chosen URL.
4. The document becomes file-backed, watching begins, and the preview rerenders with the new asset base.

## Error Handling

- A missing editor buffer fails without changing the current source.
- Save-panel cancellation is not an error and preserves the dirty draft.
- Failed writes preserve the dirty state and keep the editor/window open.
- File-backed saves retain current modified/missing/unreadable conflict handling.
- Launch URL handling must suppress untitled creation when a file-open event is pending or received.

## Accessibility and Platform Conventions

- Reuse standard AppKit document commands, `NSSavePanel`, menu shortcuts, toolbar controls, focus handling, and close confirmation.
- Keep existing localized **Untitled**, **Edited**, save-decision, and toolbar labels.
- Focus the source editor when a blank untitled document opens so keyboard and assistive-technology users can paste immediately.
- No clipboard content is read without an explicit user paste action.

## Verification

The repository has no AppKit UI test suite; its contribution guide requires manual smoke testing for UI changes. Verification will include:

1. Build the `md-preview` scheme in Debug.
2. Launch without a file and confirm one focused untitled editor opens without an open panel.
3. Paste representative Markdown containing headings, tables, task lists, code, math, Mermaid, frontmatter, and links.
4. Toggle to the full preview and back; confirm the draft and rendered features survive.
5. Save as `.md`; confirm title/metadata/actions update, the file reopens, and relative assets resolve from the saved folder.
6. Verify close-time Save, Don’t Save, and Cancel behavior for an untitled draft.
7. Verify Command-N and Command-T create untitled documents/windows in the expected placement.
8. Open and edit a file-backed document to ensure existing save and external-conflict behavior remains intact.
9. Run the existing Swift helper tests.

## Rejected Alternatives

### Preview Clipboard command

A dedicated command could render the current clipboard immediately, but it adds a second creation path, reads global clipboard state, and still needs an editing and saving lifecycle. The standard untitled-document workflow covers clipboard and typed input with less code and fewer privacy surprises. It can be reconsidered if usage shows that Command-N then Command-V is too slow.

### Split live preview

A permanent source/preview split provides immediate output but duplicates the existing edit/preview presentation, halves reading width, and continuously invokes the heavy renderer while typing. It conflicts with the app’s focused reading design and is unnecessary for the requested workflow.
