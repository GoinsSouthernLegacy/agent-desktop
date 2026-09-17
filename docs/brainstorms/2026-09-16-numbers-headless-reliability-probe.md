# Numbers headless reliability probe

Date: 2026-09-16. Machine: macOS 26.0 (25A354), Numbers 15.3.1 installed as
`/Applications/Numbers Creator Studio.app` (bundle `com.apple.Numbers`),
agent-desktop 0.9.1 release build from `feat/jev-desktop-loop` at d25e0d90.

Goal: drive the hardest first-party app end to end in strict headless mode
(no `--headed`, no cursor, no focus steal), compare every agent-desktop result
against the raw Accessibility API, and separate what Numbers cannot expose
from what agent-desktop gets wrong. The raw probes are pyobjc scripts that
call the same `AXUIElement*` functions the adapter uses; they live in
`/tmp/axprobe/` (`ax_probe.py`, `ax_probe2.py`, `chooser_probe.py`).

## Headline

The task "open Numbers, create a blank sheet, put data in a cell" has exactly
one step Numbers cannot do through accessibility (write text into a cell), and
that step is not the one that broke the agent. Every other failure the agent
hit is an agent-desktop timing or naming problem that reproduces on any app:

| Step | agent-desktop result | Raw AX truth |
|------|----------------------|--------------|
| `launch com.apple.Numbers` (with or without `--activate`) | `APP_UNRESPONSIVE` "surface read incomplete for AXWindows" after 1-3 s | `AXWindows` answers in <1 ms once the app has finished starting; the Open panel is listed 3 s later |
| `snapshot --skeleton` of the Open panel | 45 refs in 0.19 s, but 24 unnamed `treeitem`s | Every row has a `AXStaticText` grandchild with the name |
| `find --name "New Document"` + `click` | works, `delivered_unverified` | `AXPress` on the button takes 100-400 ms |
| `find --name Basic` + `click` in the template chooser | `POLICY_DENIED`, suggests `--headed` | The matched `AXCell` has no action; its parent `AXRow` is selectable |
| `click` on the "Basic" row ref | `delivered_verified` via container selection | `AXSelectedRows` = Basic |
| Locate the "Blank" tile | Not in any snapshot | Not in the AX tree at all (see A1) |
| `click` on "Create" | `APP_UNRESPONSIVE`, `AXPress` returned `kAXErrorCannotComplete` | Document was created; the press exceeded the 250 ms messaging slice |
| `snapshot --root <table>` in the document | 178 refs in 0.5 s, values as names | Matches AX; empty cells have no address in the output although AX exposes one |
| `click` on a cell | `delivered_unverified`, nothing selected | `AXPress` on a cell in an unfocused table is a no-op; `AXSelectedCells` on the table selects it |
| `set-value` / `type` on a cell | `ACTION_NOT_SUPPORTED`, "role cell is not editable" | Correct: no cell attribute is settable (see A2) |
| `set-value` / `type` on the editor a pressed cell exposes | `ACTION_FAILED`, post-state value stays empty | Correct for what was observed: the `AXTextArea` returned `ok` and changed nothing while another app was frontmost (see A2) |
| `press h`, `press i`, `press return --app Numbers` (pid-targeted, no `--headed`) | delivered | "hi" landed in the selected cell while the terminal stayed frontmost |
| `close-app` with an unsaved document | `TIMEOUT` "may be busy" | Numbers presented the save sheet; `snapshot --surface sheet` + `click Delete` quit it cleanly |

## Part A: Numbers limitations (not fixable in agent-desktop)

### A1. Template tiles are not in the accessibility tree

The chooser's grid is an `AXList` (subrole `AXCollectionList`) with 8
`AXSectionList` children. Each section reports `AXChildren`,
`AXVisibleChildren` and `AXChildrenInNavigationOrder` as empty, and the
`AXGroup` containers under them (796x282 pt, where the tiles are drawn) have
zero children. This holds with `AXEnhancedUserInterface` false, set true after
the chooser opened, and set true before the chooser opened. `AXSelectedChildren`
stays empty even while a tile is visibly selected. `AXUIElementCopyElementAtPosition`
could not be tested on the tiles without bringing Numbers to the front.

Consequence: no accessibility client can find "Blank" by name. The reachable
headless path is the sidebar (`AXTable` rows, selectable) plus "Create", which
creates whichever tile Numbers has selected. That selection is not observable
through AX (`AXSelectedChildren` stays empty while a tile is highlighted), so
an agent cannot confirm which template "Create" will use. In this probe the
user had clicked the "Categories" tile by hand, and "Create" produced that
template.

### A2. Cells accept no text write; the cell editor accepted writes and changed nothing

- `AXUIElementIsAttributeSettable` returns error -25200 for `AXValue`,
  `AXSelectedText`, `AXSelectedTextRange`, `AXFocused` and `AXTitle` on a cell,
  before and after selection. A direct `AXValue` write returns -25200.
- `AXPress` on a *selected* cell in a *focused* table enters edit mode: the cell
  title becomes `"<value>, Is Editing"` and the cell gains one `AXTextArea`
  child, which is the app's focused element and reports `AXValue`,
  `AXSelectedText` and `AXSelectedTextRange` as settable.
- Writes to that editor returned `kAXErrorSuccess` and changed nothing. Both
  raw writes and agent-desktop's `set-value` and `type` observed an empty
  value afterwards; agent-desktop reported `ACTION_FAILED` with
  `post_action_verification`, which is the correct outcome for what it saw.
- `AXReplaceRangeWithText` (parameterized) returns false. Return on a selected
  cell enters edit mode without exposing an editor.

Every editor write in this probe ran with the terminal frontmost. Whether
Numbers implements the write at all is unknown: AppKit text views route
accessibility insertion through the text input context, so a key window may
behave differently. That test needs a focus change, which this probe was asked
not to do. It matters for the fix conversation, because a key-window
requirement is exactly what the `--headed` focus elevation provides.

The one path that entered text was keystrokes. Pid-targeted `press` (CGEvent
posted to the Numbers pid) worked with another app frontmost. Whether that
counts as headless is a policy decision; the docs currently classify all
synthesized keyboard input as physical.

### A3. `AXEnhancedUserInterface` is set by something else on this machine

A fresh Numbers process reported `AXEnhancedUserInterface = true` before any
probe touched it. agent-desktop never sets it (only `AXManualAccessibility`
for Chromium). Another assistive client is the likely setter. Results on a
machine without that client may differ in element naming (with the flag on,
section lists gained `AXDescription` values such as "Basic").

The chooser sidebar changed category twice between read-only probes; the
user confirmed those were manual clicks in the Numbers window. Toggling the
flag does not move the selection.

## Part B: agent-desktop gaps (app-agnostic)

Each item lists evidence, root cause, fix direction, and what a fix could
break. Every adapter change needs the fixture E2E run and the ignored Finder
probes per `docs/solutions/best-practices/real-app-tests-are-the-platform-adapter-gate.md`.

### B1. `launch` fails during app startup instead of waiting

Evidence: three fresh launches. `launch --activate` returned
`APP_UNRESPONSIVE` (`surface_array_incomplete`, `AXWindows`, loaded 0 of 0)
after about 1 s; `launch` without `--activate` returned the same after 3.4 s;
`list-windows` showed the Open panel 3 s later each time. One slow start
(seen once) instead consumed the whole 30 s and returned `TIMEOUT`
`core_graphics_window_inventory_unstable` with `attempts: 0`; why that start
took more than 30 s to show a window is unknown.

Root cause, confirmed part: `settled_window`
(`crates/macos/src/system/launch.rs:56-80`) propagates every error from
`exact_window` except `WINDOW_NOT_FOUND`. The window read goes through
`window_ax_state::read_until` → `surface_read::elements`
(`crates/macos/src/tree/surface_read.rs:4-38`), which turns one incomplete
`AXWindows` read into `APP_UNRESPONSIVE` without the AX error code and without
the resilient retry used elsewhere (`child_source::read_resilient`, 3
attempts). The `attempts: 0` timeout comes from `stabilize_records_until`
(`crates/macos/src/system/cg_window.rs:82`), which reports "unstable" when it
is entered with an already-expired deadline.

Root cause, inferred part: the error code of the failing read is not
recorded, so why `AXWindows` fails during startup is not established. The
likely cause is the 250 ms messaging slice
(`crates/macos/src/tree/ax_ipc.rs:20,58`) expiring while the starting app is
busy; the same raw read answered in under 1 ms once the app had settled.

Fix direction: inside the launch poll loop treat `AppUnresponsive` and
`Timeout` from the AX window read as "no window yet" until the launch
deadline, and include `ax_error` in `surface_array_incomplete`. Report
deadline exhaustion as a launch timeout, not as inventory churn.

Risk: retrying must not mask a genuinely hung app; keep the final error once
the deadline expires and keep `delivered_unverified` semantics, because the
launch itself did happen.

### B2. 250 ms messaging slice makes slow-but-successful presses look uncertain

Evidence: `click` on "Create" and `select` on the sheet table both returned
`APP_UNRESPONSIVE` (`kAXErrorCannotComplete`) although the action landed. Raw
`AXPress` on the same elements with a 5 s timeout returned `ok` in 101 ms
(New Document), 34-75 ms (cell already selected) and 417 ms (cell press that
enters edit mode).

Root cause: `ax_ipc::prepare` caps every call's messaging timeout at
`MAX_IPC_SLICE` (250 ms). `perform_observed_action`
(`crates/macos/src/actions/ax_perform.rs:16`) routes through it. A button
whose handler does synchronous work (build a document, open a chooser) blocks
the reply past the slice.

Fix direction: post-state verification first. When `AXPress` returns
`CannotComplete`, poll the effect the caller can observe (new window, sheet,
focus change, selection change) before declaring delivery uncertain; the
right-click chain already re-checks state this way. A longer slice for
`perform_action` alone is secondary and must stay bounded, because a handler
that blocks on a modal sheet never returns.

Risk: widening the read slice would let a hung app consume whole deadlines;
keep reads at 250 ms.

### B3. `click` on a cell claims delivery and never selects it

Evidence: `click` on an empty sheet cell returned `AXPress succeeded,
delivered_unverified`; `AXSelectedCells` stayed empty and the screenshot showed
no selection. A raw write of `AXSelectedCells = [cell]` on the table selected
it and read back `AXSelected = true`.

Root cause: `CLICK_CHAIN` (`crates/macos/src/actions/chain_defs.rs:12-32`)
stops after the first reported success and `continue_after_unverified_delivery`
is false, so `select_within_container` never runs; and that fallback only
writes `AXSelectedRows` and `AXSelectedChildren`
(`crates/macos/src/actions/container_select.rs:24`). `AXSelectedCells` is a
standard `AXTable` attribute and is unused anywhere in the adapter.

Fix direction: verification-driven fall-through for selectable roles (`cell`,
`row`, `treeitem`, `option`, `tab`): after `AXPress`, read `AXSelected`; only
when it is still false continue to container selection, and add
`AXSelectedCells` to the container attributes.

Risk: continuing past an unverified `AXPress` is a second mutation after a
possibly delivered first; the flag was set deliberately. Gate the fall-through
on a read-back that proves the first press had no effect, and only for roles
whose "press" means "select".

### B4. `find` returns the named cell, the actionable row has no name

Evidence: in the template chooser `find --role cell --name Basic` matched the
`AXCell`; `click` on it failed the `supported_action` gate with
`POLICY_DENIED` and a suggestion to use `--headed`. The parent `AXRow` has the
`Click` capability (selection-based) and clicking its ref succeeded
`delivered_verified`. The same shape appears in the Open panel sidebar, where
`treeitem` rows *do* get names.

Root cause: child-content naming applies to `INTERACTIVE_ROLES` only
(`crates/macos/src/tree/child_labels.rs:30-32`, `crates/core/src/roles.rs:6`),
and `row` is not in that list while `treeitem` is.

Fix direction: derive a row's name from its cells the way tree items already
do, or let the `supported_action` gate report the nearest actionable ancestor
ref instead of suggesting `--headed`.

Risk: naming rows changes `find` results and golden fixtures for every table;
rows with many cells need the existing 5-child cap and the same uniqueness
rules as tree items.

### B5. Skeleton boundary nodes carry refs but no names

Evidence: the Open panel skeleton listed 24 `treeitem` refs and the chooser 9
`row` refs with no name, so an agent cannot pick which one to drill without
drilling all of them.

Root cause: the boundary child read is zero elements except at a locator root
(`crates/macos/src/tree/query/traversal.rs:88-93`). For snapshots the evidence
plan is `uniform` (`crates/core/src/live_locator/observation_request.rs:57`),
so `hydrates_root_name_from_children`
(`crates/core/src/live_locator/evidence_plan.rs:51`) is false and even the
root loads no label children.

Fix direction: extend the locator-root rule to skeleton boundary nodes whose
role names from child content, loading at most `MAX_LABEL_ELEMENTS` children
for labeling only.

Risk: boundary label reads cost one child page per boundary node; keep the
25 ms boundary count budget and mark the node truncated when the read cannot
complete, per `child_read_budget.rs`.

### B6. Empty cells have no address

Evidence: the table drill exposed empty cells only by position in the row.
Raw AX exposes `AXRowIndexRange` and `AXColumnIndexRange` on every cell (and
`AXCellForColumnAndRow` on the table). Neither attribute is read anywhere in
the adapter.

Fix direction: emit `row` and `column` indices on `cell` nodes. UIA
(`GridItem.Row/Column`) and AT-SPI (table cell position) have equivalents, so
the field can be core-level.

Risk: additive JSON field; fixtures change.

### B7. Resolution after a state change reports `AMBIGUOUS_TARGET`

Evidence: drilling the ref of a cell whose name changed from empty to
"Is Editing" returned `AMBIGUOUS_TARGET` with 38 candidates. The true target
no longer matched by name; the 38 other empty cells matched by role and name
and none matched by bounds.

Root cause: `classify_ambiguous_candidates`
(`crates/macos/src/tree/resolve_classify.rs:65`) falls through to the
ambiguous error when zero candidates match bounds.

Fix direction: zero bounds matches among name-matched candidates with a known
`bounds_hash` is `STALE_REF`, not `AMBIGUOUS_TARGET`. The recovery advice is
the same (re-snapshot) but the wrong code sends agents looking for a "more
specific ref" that cannot exist.

Risk: low; only the error code and message change.

### B8. Error text that points the wrong way

- `set-value` on a cell suggests "target the text editor it exposes" and
  "--headed alone does not enter cell editing"
  (`crates/core/src/actionability/evaluate.rs:168`). For Numbers the editor
  ignores writes, so the suggestion cannot succeed.
- `close-app` on an app with an unsaved document reports `TIMEOUT` "may be
  busy or unresponsive"; the app is waiting on a save sheet. Reporting the
  sheet (it is visible through `list-surfaces`) would let an agent handle it.
- `surface_array_incomplete` omits the AX error code, which hid the
  `CannotComplete` behind B1 for the whole first hour of this probe.

## What worked well headlessly

- Skeleton and full snapshots of AppKit panels: 0.2 s and 1.5 s.
- Drill into a 14x6 iWork table: 178 refs, 0.5 s, values as names.
- Sheet surface: `list-surfaces`, `snapshot --surface sheet`, `click` on
  "Delete" quit the app with no focus change.
- Row selection through container selection: `click` on the "Basic" row ref
  returned `delivered_verified`, and raw `AXSelectedRows` confirmed it. This is
  the proof that B4 is a naming problem only.
- Post-action verification on `set-value` and `type` caught an editor that
  accepted writes and changed nothing.
- Pid-targeted key delivery reached a background app, and `press return`
  moved the cell selection down one row. That is the commit path an agent
  needs after any fix to B3.

## Suggested order if fixing

1. B1 and B8 (launch retry, error codes and text): smallest diffs, largest
   agent-facing confusion.
2. B3 with B2's verification-first press: one change to the click chain and
   the container attributes, covered by the fixture's table scenario plus a
   new ignored Numbers probe.
3. B4 and B5 (naming): golden fixtures change; do them together.
4. B6 and B7: additive.

## Addendum, 2026-09-16: fixes applied and cross-app verification

Everything below was measured after the probe, while fixing. The desktop was
never made exclusive: every command ran headless with another application
frontmost, except two deliberate `focus-window` calls that are called out.

### A4. Numbers builds its sheet canvas only once the window has been key

A document created headlessly is visible on screen with its table drawn, and
absent from the accessibility tree: the canvas `AXScrollArea` reports only its
two scroll bars, with no `AXLayoutArea` and no `AXTable`. One `focus-window`
call makes the layout area and the table appear, and they remain after focus
moves to another application. So one activation per document unlocks headless
observation for the rest of that document's life, and without it no reader can
see the sheet at all. This is a Numbers behaviour; agent-desktop reports the
tree it is given.

### B9. A snapshot fails when an application has several unfocused windows

`snapshot --app Finder` returns `AMBIGUOUS_TARGET` when Finder has two windows
and neither is focused, and succeeds with `--window-id`. This broke the repo's
own `snapshot_resolves_a_window_id_reported_by_list_windows` probe during this
work. The accessibility API exposes `AXMainWindow` for exactly this case, and
preferring it would resolve the common one. Not fixed here; the probe's failure
is environmental, and no snapshot code was changed.

### What was fixed, and why each fix is not about Numbers

Every rule below keys on a standard accessibility attribute or on the role
vocabulary that already lived in `agent-desktop-core`. The diff contains no
application name, bundle identifier, or vendor-specific action string.

| Fix | Rule | Evidence it is general |
|-----|------|------------------------|
| B2, mutation timeout | A read gets a 250 ms slice because it is one step of a loop that re-checks its deadline; a mutation has no such loop and gets half the remaining deadline | Pure timing policy, applies to every `AXUIElementPerformAction` and `AXUIElementSetAttributeValue` on any application |
| B3, press verified by selection | For a role that activates by selection, a press that leaves `AXSelected` false is a delivery with no effect, and only that outcome lets the chain continue | Uses the pre-existing core role set (row, treeitem, cell, listitem, option, tab) and the standard `AXSelected` |
| B3, cell selection | `AXSelectedCells` added to the container selection attributes | Standard `NSAccessibility` table attribute, published by any `NSTableView` grid |
| B3, selection target | When the target is selectable in its own right, find the container that owns its selection instead of climbing to a larger element and selecting that | Role-based; prevents selecting a whole row when a cell was asked for |
| Settability probe | An unanswerable settability read means "do not attempt this write", not "fail the command" | Reuses `ax_absence::is_absent_attribute_error`, whose own comment warns against a second definition |
| Effect probe | A selection that turned on is evidence of an effect, on any element, alongside the existing focus rule | Fixes Finder, which was never part of the original report |
| Selection readback | A selection write is applied asynchronously, so the readback waits, bounded, and only after an accepted write | Matches three existing bounded polls in the same crate |

### Measured results

| Case | Before | After |
|------|--------|-------|
| Numbers, click "Create" | `APP_UNRESPONSIVE`, `kAXErrorCannotComplete`, unsafe to retry, while the document it created was on screen | `ok`, `AXPress succeeded`, 1949 ms in the lease |
| Numbers, click an empty sheet cell | `ok`, `delivered_unverified`, nothing selected | `delivered_verified`, `verified: true`, 82 to 148 ms, selection confirmed by an independent raw read |
| Finder, click a sidebar row | `ACTION_FAILED`, "AXOpen did not establish a verifiable effect", `delivery_uncertain` — while the click had in fact worked | `delivered_verified` through `activate_descendant`, selection confirmed and restored |
| Numbers, click "New Document" | `AXPress succeeded`, unverified | unchanged, which is the point: a plain button keeps today's behaviour |

Finder matters here because it is a different application, a different widget
family (`AXOutline` and `AXOutlineRow` rather than `AXTable` and `AXCell`), and
it was failing before this work for a reason the original report never saw.

### Still ungated

- `bash tests/e2e/run.sh` needs `AGENT_DESKTOP_E2E_EXCLUSIVE=1`, which its own
  README says must not be set while the desktop is in use. It has not been run.
- `bash scripts/perf-baseline-compare.sh` has not been run. The changes add at
  most two accessibility reads per performed action and one bounded wait after
  an accepted selection write.
- The three `#[ignore]` Finder probes were run: two passed, and the third
  failed for the environmental reason recorded as B9.

### Verification after the launch and resolution fixes landed

| Case | Before | After |
|------|--------|-------|
| Fresh `launch com.apple.Numbers`, run twice | `APP_UNRESPONSIVE`, `surface_array_incomplete` | `ok` with the window, 4.0 s and 1.7 s |
| Fresh `launch --activate` | `APP_UNRESPONSIVE` | `ok` with the window, 2.0 s |
| `launch` attaching to a running instance | `APP_UNRESPONSIVE` | `ok` with the window |
| Resolve a cell ref after the cell's name changed | `AMBIGUOUS_TARGET`, 38 candidates, "retry with a more specific ref" | `STALE_REF`, `kind: "bounds_mismatch"`, 39 candidates, refresh-and-retry recovery |
| The three `#[ignore]` Finder probes | 2 passed, 1 failed on B9 | all 3 passed once Finder had a resolvable window |

A launch that succeeds can still return a window with zero bounds and
`visible: false`, because the window exists before it is laid out. That is an
honest report rather than an error, but a caller that snapshots immediately may
observe an empty tree and should wait for bounds.

Two observations about the machine during this work, neither a product defect:

- Under a load average near 10, `find` returned `TIMEOUT` with
  `locator_transient_incomplete` on two attempts out of three, while a
  `snapshot` of the same window succeeded. The deadline is doing its job; the
  asymmetry between `find` and `snapshot` under load is worth a look.
- The disk filled to under 200 MB free during parallel builds, and two test
  runs failed with `No space left on device` in `trace_read::html`. Those
  failures were environmental.

### Naming verification, and the one performance cost

Skeleton mode was the whole point of B4 and B5, and it now names its drill
targets on both applications tested.

| Skeleton snapshot | Before | After |
|-------------------|--------|-------|
| Numbers Open panel | 24 `treeitem` refs, 0 named | 24 refs, 24 named |
| Finder sidebar | unnamed | 25 refs, 25 named |
| Numbers template chooser | 9 `row` refs, 0 named | 9 refs, 9 named, "All Templates" through "Education" |

`find --name "Basic"` with no role filter used to return the `AXCell`, which
advertises no action and failed the actionability gate with `POLICY_DENIED`.
It now returns the `row`, and clicking that ref reports `delivered_verified`
in 109 ms with the selection confirmed.

The cost is real and was measured. `scripts/perf-baseline-compare.sh
--skip-fixture --apps "Finder,Numbers" --rounds 8` against the merge base gave,
on Finder, with an identical node and ref count on both sides:

| Case | Base p50 | HEAD p50 |
|------|----------|----------|
| `snapshot skeleton` | 259 ms | 379 ms |
| `snapshot -i` | 3069 ms | 3079 ms |
| `snapshot d30` | 3076 ms | 3067 ms |

Only skeleton mode moved, by 46 percent, which is the boundary label read. Most
of it was waste: a scroll area, a toolbar or a group at the depth cutoff was
loading children no label would ever be read from, and so was a row that
already had its own title. A later commit gates the load on the role, and
naming was re-verified as unchanged at 24 of 24. The re-measurement of the
gated version is still outstanding: the machine went to a load average above 40
under a Time Machine backup and a disk purge, which makes any timing
meaningless.

The Numbers side of that run is not usable. Its success rate was between
0.125 and 0.375 because the application was timing out under load, so only the
Finder column supports a conclusion.
