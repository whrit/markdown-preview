# Untitled Markdown Preview Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let users create an unsaved Markdown document, paste or type source, view it through the existing full renderer, and optionally save it as a `.md` file.

**Architecture:** Keep `MarkdownDocument` and `DocumentWindowController` as the document lifecycle owners, but treat `fileURL` as optional metadata rather than a prerequisite for content. Reuse the existing CodeMirror editor and WKWebView renderer; add only the nil-URL render/save paths and standard AppKit untitled-document creation.

**Tech Stack:** Swift 6, AppKit `NSDocument`/`NSDocumentController`, WebKit, `NSSavePanel`, UniformTypeIdentifiers, existing swift-markdown/CodeMirror renderer.

## Global Constraints

- Minimum deployment target remains macOS 15.0.
- The app remains sandboxed; all new-file writes must follow user approval through `NSSavePanel`.
- Do not add dependencies, renderers, view-controller types, or clipboard-reading code.
- Reuse the existing Edit Mode, Command-E toggle, outline, inspector, export, and external-edit conflict behavior.
- Untitled drafts are in-memory only; do not add restoration or autosave persistence.
- Existing incoming-file launch behavior must not create an extra untitled window.
- UI verification is manual because this repository has no AppKit UI test target.

---

### Task 1: Make untitled documents editable and renderable

**Files:**
- Modify: `md-preview/MarkdownDocument.swift:40-95`
- Modify: `md-preview/DocumentWindowController.swift:261-285, 719-725, 872-950, 1123-1180, 1256-1280, 1401-1483, 1703-1706, 3035-3063`
- Modify: `md-preview/MainSplitViewController.swift:279-300`

**Interfaces:**
- Consumes: existing `MainSplitViewController.display(markdown:fileName:url:assetBaseURL:)` and `EditorViewController.focusEditor()`.
- Produces: `MarkdownDocument.replaceContents(markdown:fileURL:)` accepting `URL?`; `DocumentWindowController.renderCurrentDocument(text:fileURL:)` accepting `URL?`; edit/preview behavior that works when `currentFileURL == nil`.

- [ ] **Step 1: Record the failing untitled-document behavior**

Build the current app into a deterministic location:

```bash
xcodebuild -project md-preview.xcodeproj \
  -scheme md-preview \
  -configuration Debug \
  -derivedDataPath /tmp/markdown-preview-derived \
  build
```

Launch `/tmp/markdown-preview-derived/Build/Products/Debug/Markdown Preview.app`, invoke File → New, and record the current failure: the untitled window cannot enter Edit Mode or produce a rendered preview because `currentFileURL` is nil. Do not add UI-test infrastructure for this one AppKit flow.

- [ ] **Step 2: Let the document model retain source before a URL exists**

Change the existing replacement API rather than adding a parallel untitled API:

```swift
func replaceContents(markdown: String, fileURL: URL? = nil) {
    markdownStorage.withLock { $0 = markdown }
    if let fileURL {
        replaceFileURL(fileURL)
    }
}
```

Before changing the exported signature, run LSP references for `replaceContents(markdown:fileURL:)`; every existing call must continue compiling through the optional default.

- [ ] **Step 3: Render every displayed document, including untitled source**

In `DocumentWindowController.display(markdown:fileURL:)`, keep file-only recent-document, watcher, and default-handler work inside `if let fileURL`, but move rendering outside that branch:

```swift
renderCurrentDocument(text: markdown, fileURL: fileURL)
if fileURL == nil {
    enterEditMode()
}
```

Change the render helper to derive metadata conditionally:

```swift
private func renderCurrentDocument(text: String, fileURL: URL?) {
    let fileName = fileURL?.lastPathComponent
        ?? NSLocalizedString("Untitled", comment: "Window title when no document is open")
    (documentWindow.contentViewController as? MainSplitViewController)?
        .display(markdown: text,
                 fileName: fileName,
                 url: fileURL,
                 assetBaseURL: fileURL?.deletingLastPathComponent())
}
```

Update every callsite found through LSP references. File-backed calls keep passing their URL; draft-only rerenders pass the optional current URL.

- [ ] **Step 4: Remove URL gates from Edit Mode**

Use source presence as the edit invariant:

```swift
var canToggleEditMode: Bool {
    isEditing || currentMarkdown != nil
}
```

Apply the same condition to `updateEditToolbarItem()`. In `enterEditMode()`, remove `currentFileURL != nil` from the guard. In `previewPendingEdits()`, do not require a URL:

```swift
guard let self, let markdown else {
    NSSound.beep()
    return
}
self.editorDraftMarkdown = markdown
self.currentMarkdown = markdown
self.hasUnsavedEditorChanges = markdown != self.editorBaselineMarkdown
self.markdownDocument?.replaceContents(markdown: markdown, fileURL: self.currentFileURL)
```

Change draft discard, external adoption, exit rerender, and `rerenderCurrentPreview()` callsites so they update/render source with an optional URL. Keep disk-state checks conditional through the existing `diskFileState(for:expectedMarkdown:)` behavior; do not invent a fake temporary path.

- [ ] **Step 5: Focus the editor when it becomes visible**

At the point `MainSplitViewController.revealEditorIfPrepared(_:)` marks the editor visible, focus the existing editor:

```swift
self.isEditorPreparing = false
self.isEditorVisible = true
editorVC.focusEditor()
```

This makes File → New immediately ready for typing or Command-V and also improves the existing file-backed Edit action.

- [ ] **Step 6: Build and smoke-test the source-to-preview loop**

Run:

```bash
xcodebuild -project md-preview.xcodeproj \
  -scheme md-preview \
  -configuration Debug \
  -derivedDataPath /tmp/markdown-preview-derived \
  build
```

Launch the app through the harness process manager. File → New must show **Untitled** in focused Edit Mode. Paste:

````markdown
---
title: Clipboard draft
---

# Clipboard draft

- [x] Task
- **Rendered** text

```swift
print("hello")
```

$$x^2 + y^2 = z^2$$
````

Invoke Command-E. Confirm the full renderer shows frontmatter, heading outline, task, bold text, highlighted code, and math. Invoke Command-E again and confirm the exact source remains.

- [ ] **Step 7: Commit the independently working untitled preview**

```bash
git add md-preview/MarkdownDocument.swift \
  md-preview/DocumentWindowController.swift \
  md-preview/MainSplitViewController.swift
git commit -m "Add editable untitled Markdown previews"
```

---

### Task 2: Save untitled Markdown safely

**Files:**
- Modify: `md-preview/DocumentWindowController.swift:46-63, 719-725, 1112-1121, 1281-1451, 1486-1535, 1784-1832`
- Verify unchanged responder wiring: `md-preview/Base.lproj/MainMenu.xib:110-119`

**Interfaces:**
- Consumes: Task 1’s optional-URL `replaceContents(markdown:fileURL:)` and `renderCurrentDocument(text:fileURL:)`.
- Produces: `saveDocumentAs(_:)`, `presentMarkdownSavePanel(_:suggestedURL:completion:)`, and `adoptSavedMarkdown(_:fileURL:)`; Command-S and Save As work for untitled drafts.

- [ ] **Step 1: Verify the failing save contract**

With an untitled draft open, invoke Command-S. The pre-change flow reaches `saveEditedMarkdown` with no URL and returns `.cancelled`; no save panel appears. Close the dirty window and choose Save; the same failure must leave the window open. This is the behavioral failure the task fixes.

- [ ] **Step 2: Enable Save and Save As for in-memory source**

Extend menu validation without enabling unrelated document actions:

```swift
func validateMenuItem(_ menuItem: NSMenuItem) -> Bool {
    switch menuItem.action {
    case #selector(saveDocument(_:)):
        return isEditing || hasPendingEditorChanges
    case #selector(saveDocumentAs(_:)):
        return currentMarkdown != nil && !isEditorCommitInFlight
    default:
        syncSidebarMenuState()
        return true
    }
}
```

Keep `saveDocument(_:)` routed through `commitEdits(exitAfter: false)` so close/termination completions still serialize behind an in-flight commit.

- [ ] **Step 3: Add one save-panel implementation**

Add a helper that both untitled Save and Save As use:

```swift
private func presentMarkdownSavePanel(
    _ markdown: String,
    suggestedURL: URL?,
    completion: @escaping (EditedMarkdownSaveResult) -> Void
) {
    let panel = NSSavePanel()
    panel.directoryURL = suggestedURL?.deletingLastPathComponent()
    let untitledName = NSLocalizedString(
        "Untitled", comment: "Window title when no document is open")
    panel.nameFieldStringValue = suggestedURL?.lastPathComponent ?? "\(untitledName).md"
    panel.allowedContentTypes = [
        UTType("net.daringfireball.markdown"),
        UTType(filenameExtension: "md"),
    ].compactMap { $0 }
    panel.beginSheetModal(for: documentWindow) { [weak self] response in
        guard let self, response == .OK, let url = panel.url else {
            completion(.cancelled)
            return
        }
        guard self.write(markdown, to: url) else {
            completion(.cancelled)
            return
        }
        self.adoptSavedMarkdown(markdown, fileURL: url)
        completion(.saved)
    }
}
```

Use the existing `UniformTypeIdentifiers` import and write helper. Do not create a temporary file or bypass `NSSavePanel`.

- [ ] **Step 4: Adopt a successful save everywhere once**

Add the successful transition:

```swift
private func adoptSavedMarkdown(_ markdown: String, fileURL: URL) {
    currentFileURL = fileURL
    currentMarkdown = markdown
    markdownDocument?.replaceContents(markdown: markdown, fileURL: fileURL)
    documentWindow.title = fileURL.lastPathComponent
    NSDocumentController.shared.noteNewRecentDocumentURL(fileURL)
    refreshOpenWithItem()
    refreshOpenInLLMItem()
    refreshOpenActionsItem()
    startWatching(fileURL)
    renderCurrentDocument(text: markdown, fileURL: fileURL)
}
```

The rerender is required: it changes the nil asset base to the chosen file’s parent directory. Do not call `handleRename(to:)`; that method intentionally skips rerendering and currently requires an existing URL.

- [ ] **Step 5: Route untitled Save through the panel**

Replace the nil-URL cancellation at the start of `saveEditedMarkdown`:

```swift
if currentFileURL == nil {
    presentMarkdownSavePanel(text, suggestedURL: nil, completion: completion)
    return
}
```

Then unwrap the file-backed URL and preserve every existing `.unchanged`, `.modified`, `.missing`, and `.unreadable` branch unchanged.

- [ ] **Step 6: Implement Save As without a second persistence path**

Add a source-fetch helper:

```swift
private func markdownForSaving(_ completion: @escaping (String?) -> Void) {
    if isEditing, let editor = mainSplit?.editorViewController {
        editor.fetchMarkdown(completion)
    } else {
        completion(editorDraftMarkdown ?? currentMarkdown)
    }
}
```

Add the responder action used by the existing XIB menu item:

```swift
@IBAction func saveDocumentAs(_ sender: Any?) {
    markdownForSaving { [weak self] markdown in
        guard let self, let markdown else {
            NSSound.beep()
            return
        }
        self.presentMarkdownSavePanel(markdown,
                                      suggestedURL: self.currentFileURL) { result in
            guard case .saved = result else { return }
            self.editorDraftMarkdown = nil
            self.editorBaselineMarkdown = self.isEditing ? markdown : nil
            self.hasUnsavedEditorChanges = false
        }
    }
}
```

A cancelled or failed Save As must leave the original URL and dirty state untouched. `adoptSavedMarkdown` runs only after the new file is written.

- [ ] **Step 7: Smoke-test save, cancellation, close, and asset-base adoption**

Build again with the Task 1 command. Then verify all of these manually:

1. Untitled Edit Mode → paste → Command-S → cancel: source and **Edited** state remain.
2. Command-S → save as `draft.md`: title changes, **Edited** clears, file contains byte-faithful source, Open/Share/Inspector controls now have file metadata.
3. Use a relative image such as `![asset](asset.png)` beside the selected file; it begins resolving after the save rerender.
4. Modify the draft, close, choose Cancel: window remains.
5. Close again, choose Don’t Save: window closes.
6. Create another untitled draft, close, choose Save, approve the panel: window closes only after the file is written.
7. On a file-backed document, Save As creates the new file and the current window follows it.
8. Existing external-edit conflict choices still behave as before.

- [ ] **Step 8: Commit safe untitled saving**

```bash
git add md-preview/DocumentWindowController.swift
git commit -m "Save untitled Markdown documents"
```

---

### Task 3: Open untitled documents on launch and in new tabs

**Files:**
- Modify: `md-preview/AppDelegate.swift:114-188, 337-384`
- Modify: `md-preview/DocumentWindowController.swift:150-169, 2877-2910`

**Interfaces:**
- Consumes: standard `NSDocumentController.shared.newDocument(_:)` and Task 1’s untitled window behavior.
- Produces: no-file launch and reopen create one untitled document; `newWindowForTab(_:)` creates an untitled tab; incoming URLs suppress scheduled untitled creation.

- [ ] **Step 1: Preserve the launch-race invariant while changing the action**

Keep the existing asynchronous generation guard, but rename it around untitled creation rather than an open panel:

```swift
private var isUntitledDocumentScheduled = false
private var untitledDocumentScheduleGeneration = 0

private func scheduleUntitledDocument(requiresNoDocuments: Bool = false) {
    guard !isUntitledDocumentScheduled else { return }
    isUntitledDocumentScheduled = true
    untitledDocumentScheduleGeneration += 1
    let generation = untitledDocumentScheduleGeneration
    DispatchQueue.main.async { [weak self] in
        guard let self,
              self.untitledDocumentScheduleGeneration == generation else { return }
        self.isUntitledDocumentScheduled = false
        guard !requiresNoDocuments || NSDocumentController.shared.documents.isEmpty else { return }
        NSApp.activate(ignoringOtherApps: true)
        NSDocumentController.shared.newDocument(nil)
    }
}

private func cancelScheduledUntitledDocument() {
    guard isUntitledDocumentScheduled else { return }
    untitledDocumentScheduleGeneration += 1
    isUntitledDocumentScheduled = false
}
```

Remove launch-only open-panel state that no longer has a caller: `isOpeningDocumentFromPrompt`, `documentPromptScheduleGeneration`, `isDocumentPromptScheduled`, and `activeOpenPanel`. Remove the dead assignments around the explicit open completion. Keep `isPromptingForDocument` for explicit Command-O serialization. Keep `applicationShouldOpenUntitledFile` returning `false`; this app creates the document through the guarded custom schedule so AppKit cannot race a second untitled document against incoming URL events.

- [ ] **Step 2: Replace no-file prompt callsites**

Update these paths to call `scheduleUntitledDocument(requiresNoDocuments: true)`:

- `applicationDidFinishLaunching` when no URL arrived.
- `applicationOpenUntitledFile`.
- `applicationShouldHandleReopen` when no window is visible.
- the final failed incoming-URL completion when no documents opened.

At the start of `application(_:open:)`, call `cancelScheduledUntitledDocument()` before opening URLs. Do not change explicit `openDocument(_:)` or `promptForDocument()`; Command-O still presents the file/folder panel.

- [ ] **Step 3: Make Command-T create an untitled tab**

Replace the file prompt in `DocumentWindowController.newWindowForTab(_:)`:

```swift
override func newWindowForTab(_ sender: Any?) {
    Self.markNextWindowAsTab()
    NSDocumentController.shared.newDocument(sender)
}
```

Keep project-navigator **Open in New Tab** routed through `openInNewTab(_:)` so a selected file still opens as a tab.

- [ ] **Step 4: Build and verify launch/new behavior end to end**

Run:

```bash
xcodebuild -project md-preview.xcodeproj \
  -scheme md-preview \
  -configuration Debug \
  -derivedDataPath /tmp/markdown-preview-derived \
  build
```

Verify:

1. Cold launch with no URL opens exactly one focused untitled editor and no open panel.
2. Cold launch by opening `samples/full.md` opens that file and no untitled window.
3. Command-N creates an untitled document with native tab/window placement according to macOS tab preferences.
4. Command-T creates an untitled tab in the active document window.
5. Closing the last window and clicking the Dock icon creates a new untitled editor.
6. Command-O still opens the existing Markdown file/folder panel.
7. A failed incoming file open falls back to one untitled editor after presenting the error.

- [ ] **Step 5: Run repository regressions**

Run the existing helper suite:

```bash
swift test --package-path tests/swift-tests
```

Then request workspace LSP diagnostics for all changed Swift files. Expected: zero new errors or warnings attributable to the feature.

- [ ] **Step 6: Commit launch and tab behavior**

```bash
git add md-preview/AppDelegate.swift md-preview/DocumentWindowController.swift
git commit -m "Open untitled Markdown drafts by default"
```

---

## Final Acceptance Checklist

- [ ] No-file launch, Command-N, and Command-T produce focused untitled editors.
- [ ] Pasted source toggles to the complete existing renderer without a file.
- [ ] Draft source survives repeated Edit/Preview transitions byte-for-byte.
- [ ] Command-S and Save As produce `.md` source through `NSSavePanel`.
- [ ] Cancelled/failed saves never clear the draft or dirty state.
- [ ] Save/Don’t Save/Cancel close behavior prevents data loss.
- [ ] Saving adopts the URL and immediately enables relative assets and file-backed actions.
- [ ] Incoming file launches, explicit Open, file watching, conflict handling, export, Quick Look, and file-backed editing remain unchanged.
- [ ] Debug build and existing Swift helper tests pass.
