# tmux / vim

Reach for this for terminal-multiplexer and editor survival shortcuts.

!!! note "Stub - scaffolded, not yet filled"
    Section skeleton below. Fill over time.

## tmux - sessions

- `tmux new -s name`, `tmux attach -t name`, `tmux ls`
- detach `Ctrl-b d`

## tmux - windows & panes

- window: `Ctrl-b c` new, `Ctrl-b n/p` next/prev
- pane: `Ctrl-b %` vsplit, `Ctrl-b "` hsplit, `Ctrl-b arrow` move
- `Ctrl-b z` zoom pane, `Ctrl-b x` kill pane

## vim - modes & movement

- modes: normal / insert (`i`) / visual (`v`) / command (`:`)
- move: `w b e`, `0 $`, `gg G`, `:N`

## vim - editing

- `dd` delete line, `yy` yank, `p` paste, `u` undo, `Ctrl-r` redo
- `ciw` change word, `dt<char>` delete till

## vim - search & replace

- `/pattern`, `n/N`, `:%s/old/new/g`

## Gotchas / things I always forget

## Quick reference

| Task | Keys |
|---|---|
| tmux detach | `Ctrl-b d` |
| tmux split | `Ctrl-b %` / `"` |
| vim save+quit | `:wq` |
| vim quit no save | `:q!` |
| vim replace all | `:%s/a/b/g` |
