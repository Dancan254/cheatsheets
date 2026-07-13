# Bash

Reach for this for the scripting syntax you always forget - the stuff between "I know
bash" and "why won't this work".

## Script header (always)

```bash
#!/usr/bin/env bash
set -euo pipefail       # exit on error, undefined var = error, pipe fails if any part fails
IFS=$'\n\t'             # sane word-splitting (spaces stop breaking loops)
```

`set -e` exit on any command failure. `set -u` error on unset variable. `set -o pipefail`
a pipeline fails if *any* stage fails, not just the last. `set -x` to trace (debug).

## Variables

```bash
name="value"            # NO spaces around =
echo "$name"            # always quote - prevents word-splitting/globbing
readonly PI=3.14
local x=1               # inside functions only

count=$((3 + 4))        # arithmetic
count=$(( count + 1 ))
```

## Parameter expansion (the good stuff)

```bash
${var:-default}         # use default if var unset/empty
${var:=default}         # assign default if unset
${var:?error msg}       # error out if unset (great with set -u)
${var:+alt}             # use alt only if var IS set
${#var}                 # length
${var:2:3}              # substring (offset 2, length 3)
${var/foo/bar}          # replace first foo→bar
${var//foo/bar}         # replace all
${var%.txt}             # strip shortest suffix match  (file.txt → file)
${var%%.*}              # strip longest suffix
${var#*/}               # strip shortest prefix
${var##*/}              # strip longest prefix (→ basename)
${var,,}                # lowercase;  ${var^^} uppercase
```

## Conditionals

```bash
if [[ $x -eq 5 ]]; then echo eq; elif [[ $x -gt 5 ]]; then echo gt; else echo lt; fi

# numeric: -eq -ne -lt -le -gt -ge
# string:  = != <  >     -z (empty)  -n (non-empty)
# files:   -f (file) -d (dir) -e (exists) -r -w -x -s (non-empty)

[[ -f config.yml ]] && echo "exists"
[[ -z "$var" ]] && echo "empty"
[[ "$name" == a* ]] && echo "starts with a"     # glob match
[[ "$name" =~ ^[0-9]+$ ]] && echo "all digits"  # regex (=~)

# combine
[[ $x -gt 0 && $x -lt 10 ]]
[[ $a == "x" || $b == "y" ]]
```

Use `[[ ]]` not `[ ]` - safer (no word-split surprises), supports `&&`, `||`, `=~`, glob.

## Loops

```bash
for i in 1 2 3; do echo "$i"; done
for i in {1..10}; do echo "$i"; done
for i in {0..20..5}; do echo "$i"; done         # step by 5
for ((i=0; i<10; i++)); do echo "$i"; done
for f in *.txt; do echo "$f"; done              # globs

while [[ $n -lt 5 ]]; do ((n++)); done
until [[ $done == true ]]; do ...; done

# read a file line by line (the correct way)
while IFS= read -r line; do
  echo "$line"
done < file.txt
```

## Arrays

```bash
arr=(a b c)
arr+=(d)                # append
echo "${arr[0]}"        # first
echo "${arr[@]}"        # all elements
echo "${#arr[@]}"       # count
for x in "${arr[@]}"; do echo "$x"; done

# associative (dict)
declare -A map
map[key]="value"
echo "${map[key]}"
for k in "${!map[@]}"; do echo "$k=${map[$k]}"; done
```

## Functions

```bash
greet() {
    local name="$1"                 # positional args: $1 $2 ...  $@ all  $# count
    echo "Hi $name"
    return 0                        # 0 = success; capture output with $(greet x)
}
greet "world"
result=$(greet "world")
```

## Command substitution & redirection

```bash
today=$(date +%F)
files=$(ls *.txt)

cmd > out.txt           # stdout → file (truncate)
cmd >> out.txt          # append
cmd 2> err.txt          # stderr → file
cmd > all.txt 2>&1      # both → file  (order matters!)
cmd &> all.txt          # both → file (bash shorthand)
cmd 2>/dev/null         # discard stderr
cmd1 | cmd2             # pipe
diff <(cmd1) <(cmd2)    # process substitution - treat output as a file
```

## Traps (cleanup on exit)

```bash
tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT          # runs on any exit - cleanup guaranteed
trap 'echo "interrupted"; exit 1' INT TERM
```

## Args & guards

```bash
[[ $# -lt 1 ]] && { echo "usage: $0 <arg>"; exit 1; }
input="${1:?need an argument}"

command -v jq >/dev/null || { echo "jq required"; exit 1; }   # check a dep exists
```

## Gotchas / things I always forget

- **Quote your variables.** `"$var"` not `$var`. Unquoted, a value with spaces splits into multiple args and globs expand.
- `[[ ]]` over `[ ]` - the single-bracket `test` word-splits and needs quoting gymnastics.
- No spaces around `=` in assignment (`x=1`), but you **need** spaces inside `[[ x -eq 1 ]]`.
- `set -e` doesn't trigger inside `if`/`&&`/`||` conditions, or for the non-last command in a pipe (that's what `pipefail` fixes).
- `$(...)` over backticks - nestable and readable.
- Reading a file with `for line in $(cat f)` splits on spaces, not lines. Use `while IFS= read -r line`.
- `read -r` always - without `-r`, backslashes get mangled.
- `2>&1` must come **after** `> file` to redirect both; `2>&1 > file` sends stderr to the old stdout (terminal).
- Loops in a pipeline (`cmd | while read`) run in a **subshell** - variables set inside don't survive. Use process substitution: `while read ...; done < <(cmd)`.
- `trap ... EXIT` is how you guarantee temp-file cleanup even on failure.

## Quick reference

| Task | Snippet |
|---|---|
| Strict mode | `set -euo pipefail` |
| Default value | `${var:-default}` |
| basename | `${path##*/}` |
| strip extension | `${file%.*}` |
| Array length | `${#arr[@]}` |
| Read file lines | `while IFS= read -r l; do ...; done < f` |
| Both streams to file | `cmd &> out` |
| Cleanup on exit | `trap 'rm -f "$tmp"' EXIT` |
| Require arg | `${1:?usage}` |
| Dep check | `command -v tool >/dev/null` |
