# grep / sed / awk / find

Reach for this for text surgery on the command line.

!!! note "Stub - scaffolded, not yet filled"
    Section skeleton below. Fill with one-liners over time.

## grep - search

- `-r` recursive, `-n` line numbers, `-i` case-insensitive
- `-E` extended regex, `-o` only match, `-v` invert, `-c` count
- `-A/-B/-C N` context lines

## sed - stream edit

- `s/old/new/g` substitute
- in-place `-i`, delete lines, print ranges

## awk - columnar / field processing

- `{print $1, $3}`, field separator `-F`, `NR`/`NF`, sum a column

## find - locate files

- `-name`, `-type f/d`, `-mtime`, `-size`, `-exec ... {} \;`

## xargs - pipe into commands

## cut / sort / uniq / tr / wc (the supporting cast)

## Gotchas / things I always forget

## Quick reference

| Task | Command |
|---|---|
| Recursive search | `grep -rn "text" .` |
| Replace in file | `sed -i 's/a/b/g' f` |
| Print column | `awk '{print $2}'` |
| Files by age | `find . -mtime -1` |
| Count occurrences | `grep -c x f` |
| Unique + counts | `sort \| uniq -c` |
