# Neon tmux Debugging

Use a real terminal session for live TUI work. Prefer `tmux` plus the LLVM backend.

## Why LLVM

Neon's TUI loop depends on `stdlib/term.read_event_async()`, which uses `blocking { ... }`.

That path is supported on native/LLVM backends, but not on the VM backend.

For live TUI debugging, use:

```bash
cd /home/zov/projects/surge/surge/neon
surge run --backend=llvm examples/menu_demo.sg
```

You can also run the built demo directly after compilation:

```bash
cd /home/zov/projects/surge/surge/neon
./target/debug/menu_demo
```

## Recommended tmux Workflow

Create a session:

```bash
cd /home/zov/projects/surge/surge/neon
tmux new -s neon-debug
```

Run the demo inside the session:

```bash
surge run --backend=llvm examples/menu_demo.sg
```

Useful keys:

- `Up` / `Down`: move selection
- `Esc`: quit the demo
- `Ctrl-b d`: detach from tmux

If you want to reopen the session later:

```bash
tmux attach -t neon-debug
```

If you want to close it completely:

```bash
tmux kill-session -t neon-debug
```

## Notes

- Always compile first when validating changes:

```bash
cd /home/zov/projects/surge/surge/neon
surge build main.sg
surge build examples/menu_demo.sg
```

- `frames/` is treated as a debug artifact and is ignored by git.
- If runtime behavior looks impossible, suspicious, or compiler-driven, stop and investigate before stacking more changes on top.
