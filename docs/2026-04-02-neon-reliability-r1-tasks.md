# Neon Reliability R1 Task Breakdown

**Goal:** Stabilize Neon's current TUI core so new widgets and rendering features can be added on top of a predictable event loop, resize model, and debug workflow.

**Approach:** Fix the current language hygiene issues first, then make the app loop behavior explicit and resize-safe, then remove state mismatches between view/update, and finally lock down the development workflow for live TUI debugging. Keep the scope narrow: no new widgets, no layout engine, no ANSI styling expansion beyond what is necessary to make the current core trustworthy.

**Skills:** `@task-breakdown`, `@surge-language`, `@writing-clearly-and-concisely`, `@code-review-expert`

**Tech Details:** Surge modules, `surge diag`, `surge run --backend=llvm`, `tmux`, module manifest resolution via `surge.toml`, stdlib terminal runtime via `stdlib/term`, snapshot-style verification with captured frames.

---

### Task 1: Remove Current Language Warnings And Make Control Flow Explicit

**Files:**
- Modify: `tui/event.sg`
- Modify: `tui/utils/ansi.sg`
- Modify: `examples/menu_demo.sg`
- Verify: `main.sg`

**Step 1: Write down the failing diagnostic baseline**

Run: `surge diag .`

Expected:
- Warnings `SEM3135` appear in `tui/event.sg`
- Warnings `SEM3135` appear in `tui/utils/ansi.sg`
- Warnings `SEM3135` appear in `examples/menu_demo.sg`

**Step 2: Replace legacy implicit block values with explicit `ret` or `return`**

Make these changes:
- In `tui/event.sg`, make every `compare` arm that currently returns an `Event?` value use explicit `ret`.
- In `tui/utils/ansi.sg`, make `compare r` in `utf8_encode_cp` use explicit block exits.
- In `examples/menu_demo.sg`, make the nested `compare` arms in `update()` use explicit block exits instead of relying on the legacy implicit value rule.

Implementation notes:
- Do not change behavior in this task.
- Keep each edit local to expression-valued blocks.
- Prefer the modern Surge style consistently across all touched blocks.

**Step 3: Run diagnostics again**

Run: `surge diag .`

Expected:
- No `SEM3135` warnings remain.
- Any remaining output is informational only, or zero diagnostics.

**Step 4: Smoke-test unchanged behavior**

Run: `surge run main.sg`

Expected:
- The same two `Rect` lines print as before.

**Step 5: Review the diff**

Review focus:
- No semantic changes hidden inside syntax cleanup.
- No new warnings introduced.
- Explicit exits are easy to read and consistent.

---

### Task 2: Make The App Loop Contract Explicit

**Files:**
- Modify: `tui/app.sg`
- Modify: `tui/event.sg`
- Verify: `examples/menu_demo.sg`

**Step 1: Document the current behavior before editing**

Read:
- `tui/app.sg`
- `tui/event.sg`

Write down the current facts:
- `AppCmdEnum::Quit` is honored.
- `AppCmdEnum::None` and `AppCmdEnum::Redraw` are not meaningfully distinguished in the loop.
- The loop renders after every delivered event.

**Step 2: Decide and encode the command semantics**

Target contract:
- `None`: do not redraw.
- `Redraw`: redraw once.
- `Quit`: exit cleanly.

Make these code changes:
- In `tui/app.sg`, restructure the main loop so redraw work only happens when the command requires it.
- Keep the first render path intact.
- Keep `capture()` aligned with actual redraws so frame indexes do not advance on skipped redraws.

**Step 3: Preserve simple behavior for the existing demo**

Check `examples/menu_demo.sg` and ensure:
- `Up`, `Down`, and `Resize` still request redraw.
- `Tick` can remain `None`.
- `Esc` and `Enter` still quit.

**Step 4: Verify diagnostics and execution**

Run: `surge diag .`

Expected:
- No errors.

Run: `cd /home/zov/projects/surge/surge && surge run --backend=llvm neon/examples/menu_demo.sg`

Expected:
- The demo starts.
- Arrow keys still move selection.
- `Esc` exits.

**Step 5: Review for loop correctness**

Review focus:
- No render path is skipped accidentally after startup.
- Exit paths still restore terminal state.
- Frame capture and redraw count stay in sync.

---

### Task 3: Make Resize Handling Real Instead Of Cosmetic

**Files:**
- Modify: `tui/app.sg`
- Modify: `examples/menu_demo.sg`
- Verify: `tui/utils/buffer.sg`

**Step 1: Capture the current gap**

Read:
- `tui/app.sg`
- `tui/utils/buffer.sg`

Current issue to preserve in notes:
- `Resize` events exist, but the app loop does not resize buffers when terminal dimensions change.

**Step 2: Implement resize-safe buffer lifecycle**

In `tui/app.sg`:
- On `Resize(w, h)`, refresh the active buffer dimensions before rendering the next frame.
- Reinitialize `prev` and `next` in a way that avoids diffing buffers of stale size.
- Keep the implementation simple and deterministic; a full-screen redraw after resize is acceptable.

Implementation rule:
- Do not try to preserve old cell contents across resize in `R1`.
- Prefer correctness over optimization.

**Step 3: Make the behavior observable**

Manual verification in `tmux`:
1. Start a session: `cd /home/zov/projects/surge/surge && tmux new -s neon-r1`
2. Run: `surge run --backend=llvm neon/examples/menu_demo.sg`
3. Resize the pane or split the window.

Expected:
- No crash.
- No stale render corruption.
- The list rerenders to the new pane size.

**Step 4: Re-run diagnostics**

Run: `surge diag .`

Expected:
- No new warnings or errors from the resize work.

**Step 5: Review failure cases**

Review focus:
- Zero or tiny terminal sizes do not panic.
- Buffer allocation order is straightforward.
- The first frame after resize is a clean redraw, not a partial diff against invalid dimensions.

---

### Task 4: Remove The Viewport Mismatch Between Update And View

**Files:**
- Modify: `examples/menu_demo.sg`
- Modify: `tui/widgets/list.sg`
- Verify: `tui/app.sg`

**Step 1: Record the current mismatch**

Current problem:
- `examples/menu_demo.sg` uses `viewport = 20` in `update()`.
- `tui/widgets/list.sg` uses the real frame height in `list_view()`.
- Selection movement and rendering can therefore diverge after resize or on short panes.

**Step 2: Add explicit viewport state to the demo**

In `examples/menu_demo.sg`:
- Extend `AppState` with the last known viewport height.
- Update that state on first render and on `Resize`.
- Use the stored viewport when calling `list_move()`.

Design rule:
- Keep this fix local to the demo and current list API in `R1`.
- Do not introduce a global layout engine or a generic measurement framework yet.

**Step 3: Harden `list_move()` against bad viewport input**

In `tui/widgets/list.sg`:
- Guard against `viewport <= 0`.
- Make scroll math safe when the viewport is tiny.
- Keep behavior deterministic for empty lists.

**Step 4: Verify behavior in a small terminal**

Manual verification:
1. Open the demo in `tmux`.
2. Shrink the pane height so only one or two rows are visible.
3. Press `Up` and `Down`.

Expected:
- Selected item stays visible.
- Scroll position updates predictably.
- No negative scroll or off-by-one behavior appears.

**Step 5: Review state ownership**

Review focus:
- The viewport source of truth is obvious.
- `view()` and `update()` no longer disagree about visible rows.
- There is no hidden dependency on a magic constant.

---

### Task 5: Stabilize The Debug Workflow And Artifact Hygiene

**Files:**
- Modify: `.gitignore`
- Verify: `surge.toml`
- Create: `docs/tmux-debugging.md`
- Verify: `examples/menu_demo.sg`

**Step 1: Decide artifact policy**

Choose one and implement it:
- Option A: `frames/` is a debug artifact and must be ignored by git.
- Option B: frame snapshots are part of an intentional test harness and must live under a dedicated tracked directory.

Recommended for `R1`:
- Use Option A.

**Step 2: Stop accidental confusion from `surge.toml`**

The current manifest points `[run].main` to `main.sg`, which is useful for the package but confusing for demo work.

Make the workflow explicit:
- Keep `surge.toml` unchanged unless there is a strong reason to change package semantics.
- Instead, document the required demo invocation from the parent directory:
  `cd /home/zov/projects/surge/surge && surge run --backend=llvm neon/examples/menu_demo.sg`

If a manifest change is still desired later, defer it to a separate task so it does not mix with runtime fixes.

**Step 3: Write the debugging note**

Create `docs/tmux-debugging.md` with:
- The exact `tmux` startup command.
- The required LLVM backend note.
- The reason VM is unsuitable for this TUI path because `stdlib/term.read_event_async()` uses `blocking`.
- The expected exit keys for the current demo.

**Step 4: Verify the documented workflow**

Run:
`cd /home/zov/projects/surge/surge && tmux new -d -s neon-check 'surge run --backend=llvm neon/examples/menu_demo.sg'`

Then verify manually:
- The session starts.
- The demo is visible.
- `Esc` exits.

Cleanup:
- `tmux kill-session -t neon-check`

**Step 5: Review repository hygiene**

Review focus:
- Debug artifacts do not pollute `git status`.
- The manifest still describes the package clearly.
- A new contributor can launch the demo without rediscovering the backend trap.

---

### Task 6: Final R1 Validation Pass

**Files:**
- Verify: `tui/app.sg`
- Verify: `tui/event.sg`
- Verify: `tui/widgets/list.sg`
- Verify: `examples/menu_demo.sg`
- Verify: `docs/tmux-debugging.md`

**Step 1: Run the full diagnostic pass**

Run: `surge diag .`

Expected:
- No errors.
- No `SEM3135` warnings.

**Step 2: Run the package smoke test**

Run: `surge run main.sg`

Expected:
- The basic package entrypoint still works.

**Step 3: Run the live demo through the intended workflow**

Run from parent repo:
- `cd /home/zov/projects/surge/surge`
- `surge run --backend=llvm neon/examples/menu_demo.sg`

Manual checks:
- Initial frame renders.
- `Up` and `Down` move the selection.
- Resize does not corrupt the screen.
- `Esc` exits cleanly.

**Step 4: Check repository cleanliness**

Run: `git status --short`

Expected:
- Only intentional tracked edits remain.
- No transient `frames/` noise appears.

**Step 5: Review against the R1 goal**

R1 is complete only if all are true:
- Syntax is updated for the latest Surge expectations.
- App loop semantics are explicit.
- Resize is functionally handled.
- Viewport state is coherent.
- Debug workflow is reproducible.
- No new feature surface was added beyond reliability needs.
