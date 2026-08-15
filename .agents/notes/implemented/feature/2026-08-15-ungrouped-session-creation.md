# Agent Note: Ungrouped session creation from the resident bucket

Status: implemented

English | [中文](2026-08-15-ungrouped-session-creation.zh.md)

## Problem

The sidebar's grouped view derived the Ungrouped bucket only while it held loose Sessions, so once Workspaces existed and every Session was accounted, no surface could start a Session outside every Workspace. The hero composer made such Sessions unusable even when they existed by other means: a blank Session with no owning Workspace rendered the "Choose workspace" chip placeholder and a read-only textarea, so the user was forced to pick a Workspace before typing.

## Decision

**The Ungrouped bucket is always present as the tree's trailing group.** `deriveGroups` no longer withholds it when empty, so the grouped view always offers its ＋. The grouped empty state ("No sessions yet") is gone because the bucket header replaces it; the flat list keeps its own empty state.

**The bucket's ＋ starts a loose Session.** The new `WorkspaceRuntime.startUngroupedSession` action reuses an existing loose blank Session (blank, unarchived, outside every Workspace account) or calls `session.create({})` — the Host's existing no-workspace create, which falls back to the Host cwd and attaches no account — then opens it, mirroring `connectWorkspace`'s reuse-or-create shape and non-fatal failure posture. The action reaches the browser through the new `WorkspaceBrowserInjected.startUngroupedSession`; the `SessionsPort` create face is widened to allow an omitted Workspace, and the `TestWorkspaces` double records the action.

**An ungrouped Session keeps the hero composer live.** `ConversationRoot`'s chip-label derivation now resolves a ready Workspace list with no owning Workspace to the ungrouped bucket label instead of the "Choose workspace" placeholder, so `inert` no longer applies and the textarea accepts input. The chip still opens the picker; picking a Workspace navigates to that Workspace's blank Session (the existing `selectWorkspace` semantics — there is no attach operation, see Consequences).

## Alternatives considered

**Show the cwd basename on the chip.** Rejected: the sidebar groups such Sessions under the Ungrouped bucket, and the deleted-Workspace case deliberately never shows the dead folder's name — the bucket label is cwd-independent and honest for both.

**Keep the placeholder and require a Workspace pick.** Rejected: the story is explicitly an ungrouped Session, and the composer has no other reason to be inert for it.

**Add a Host attach operation so picking a Workspace adopts the current Session.** Rejected: the picker chip's established semantics retarget the New Session flow, and adoption is a separate product gap (recorded in the ui-workspace README's Known Limitations).

## Consequences

The grouped sidebar always ends with an Ungrouped header whose ＋ starts a Session outside every Workspace; a Session born there is immediately chat-able with the bucket label on its hero chip. Deleting a Workspace now also leaves its Sessions directly usable instead of forcing a pick. The grouped empty state is gone (the flat list keeps "No sessions yet"). Loose Sessions still have no adoption entry point: the chip's pick navigates to the picked Workspace's blank Session rather than attaching the current one. The grouped empty-state branch and its coverage were removed together with the bucket's residency.
