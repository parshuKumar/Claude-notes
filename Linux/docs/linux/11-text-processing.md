# 11 — Text Processing Powertools

## ELI5 — The Simple Analogy

Imagine a **factory assembly line for words**.

A truck dumps a giant pile of raw material (your log file) onto the start of the belt. Along the belt sit specialist workers, each doing exactly **one** job and passing the result to the next:

- **`grep`** is the **quality inspector**. She stands over the belt and lets through only the items that match her template ("keep everything stamped ERROR; throw the rest in the bin"). She never modifies an item — she only keeps or rejects.
- **`sed`** is the **stamping machine**. Every item that passes underneath gets a sticker slapped on or a label peeled off ("find every '/old/' and replace it with '/new/'"). It transforms as it flows.
- **`awk`** is the **skilled machinist with a ruler**. It picks up each item, measures its columns, does arithmetic ("add up column 9 across all 40,000 items and tell me the total"), and can remember running totals in its head.
- **`cut`** is the **guillotine** — it slices each item at a fixed position and keeps one slab.
- **`sort`** dumps everything into bins in order; **`uniq`** stands right after `sort` and says "you already gave me that one, skip it."
- **`tr`** is a **bulk find-and-replace stencil** for single characters.
- **`xargs`** is the **loading crane** that scoops up whatever came off the belt and feeds it as raw material into a *different* machine that doesn't know how to grab from the belt itself.

The genius of the factory: none of these workers know about each other. They each read from the belt (stdin) and write to the belt (stdout). You **compose** them with pipes (`|`) into a custom machine for whatever question you have. That composition — small tools, one job each, plain text between them — is the entire Unix philosophy.

---

## Where This Lives in the Linux Stack

```
Hardware (SSD holding your 4 GB access.log)
  └── KERNEL
       │    • pipe() — an in-kernel 64 KB ring buffer connecting two processes
       │    • the scheduler — runs both sides of a pipe CONCURRENTLY (streaming, not batch)
       │
       └── System Calls
            │    read(0, ...)   — every tool pulls from fd 0 (stdin)
            │    write(1, ...)  — every tool pushes to fd 1 (stdout)
            │    pipe2()        — the shell creates the tube between them
            │
            └── C Standard Library (regex compiled by regcomp(); getline() reads lines)
                 │
                 └── Shell (bash — builds `grep x | sort | uniq -c` into a pipeline)
                      │
                      └── COMMANDS: grep, sed, awk, cut, sort, uniq, tr, xargs  ◀◀◀ THIS TOPIC
                                    tee, comm, join, paste, jq
```

**The key idea:** every tool in this topic reads **line-oriented plain text from stdin** and writes **line-oriented plain text to stdout**. That shared contract — established in topic 12 (Pipes & Redirection), which this topic leans on constantly — is what lets you snap them together like Lego.

---

## What Is This?

These are the tools that **search, filter, slice, transform, and summarize text** — one line at a time, in streaming pipelines. `grep` finds lines, `sed` edits them, `awk` computes over their columns, and `sort`/`uniq`/`cut`/`tr`/`xargs` reshape the stream around them.

On a production Linux box, roughly everything you care about is text: log files, config files, `/proc`, the output of `ps` and `ss` and `df`. Mastering these tools is the difference between *"I'll download the log and open it in a GUI"* and *"give me the top 10 IPs hammering us, right now, in one line."*

---

## Why Does This Matter for a Backend Developer?

| If you don't understand this... | This will happen... |
|---|---|
| `grep -A/-B/-C` context flags | You'll find the "Error" line but not the 30-line stack trace under it, and debug blind |
| `grep` is greedy, and BRE vs ERE | Your regex matches way more than you meant, or you'll paste an `egrep` pattern into `grep` and it silently won't match |
| `awk` can sum and group | You'll export a log to a spreadsheet to answer "how many 500s per endpoint" — a question `awk` answers in one line in 200ms |
| `uniq` only collapses **adjacent** dupes | You'll run `uniq` on unsorted data, get a nonsense count, and report the wrong number in the incident channel |
| `sort` is lexical by default | You'll sort response times and see `1000ms` rank *below* `9ms` because "1" < "9" as text — and misjudge the slowest endpoint |
| GNU `sed -i` vs BSD `sed -i` | Your fix script works on the Linux server but on your Mac it creates a garbage backup file (or errors), and you ship the wrong thing |
| `cut` can't handle variable whitespace | `ps aux \| cut -d' ' -f2` returns blanks and you can't figure out why, while `awk '{print $2}'` just works |
| `xargs` and `find -print0` | `find ... \| xargs rm` deletes the wrong files the moment a filename has a space in it |

---

## The Physical Reality — What a Pipeline Actually Is

When you type `grep ERROR app.log | sort | uniq -c`, the shell does **not** run grep to completion, save a temp file, then run sort. It builds all three processes **at once**, wired together by kernel pipes, and they run **concurrently** and **stream**:

```
                 kernel pipe                 kernel pipe
                 (64 KB buffer)              (64 KB buffer)
  ┌────────┐  fd1 →│▓▓▓▓│→ fd0  ┌────────┐  fd1 →│▓▓▓▓│→ fd0  ┌─────────┐
  │  grep  │───────┼────┼───────│  sort  │───────┼────┼───────│ uniq -c │──→ TTY
  └────────┘       └────┘       └────────┘       └────┘       └─────────┘
   reads app.log                (must buffer ALL              streams out
   line by line,                input — sort can't            as sort feeds it
   emits matches                emit until it has
   immediately                  seen every line)
```

Consequences you must internalize:

- **grep and uniq stream** — O(1) memory, emitting as they go. `grep ERROR huge.log | head` *stops early* (grep gets `SIGPIPE` when head closes the pipe), so it's instant even on a 40 GB file.
- **sort must buffer everything** — it can't emit the first sorted line until it's read the *last* input line; on huge files it spills to `/tmp`. This is the one non-streaming link and usually the pipeline's bottleneck.
- **If a buffer fills, the writer blocks** — backpressure is automatic and free (the kernel sleeps grep until sort drains the pipe).

---

## How It Works — Step by Step

### Trace: `grep ERROR app.log`

```
COMMAND:     grep ERROR app.log
SHELL DOES:  fork() + execve("/usr/bin/grep", ["grep","ERROR","app.log"])
GREP DOES:   regcomp("ERROR")  → compiles the pattern into a state machine ONCE
SYSCALLS:    openat("app.log", O_RDONLY) = 3
             loop:
               read(3, buf, 32768)          ← pull a chunk
               (split buf into lines, run each through the compiled matcher)
               write(1, "…ERROR…\n", n)     ← emit only matching lines
             read(3, "", 32768) = 0         ← EOF
             close(3)
EXIT CODE:   0 if ≥1 line matched, 1 if none matched, 2 on error
             ↑ THIS is why `if grep -q X f; then` works as a test
```

### Trace: how `awk '{sum+=$3} END{print sum}'` runs

```
For EACH input line (record):
    1. split on FS (default: whitespace runs) → $1, $2, $3, …; NF, NR set; $0 = whole line
    2. run main action:  sum += $3   (numeric value of field 3 added to sum)
At EOF:
    3. run END action:   print sum
```

`awk` is a full language with an **implicit read-line loop** wrapped around your `{ action }`. That loop, plus auto field-splitting, plus associative arrays, is why it eats structured logs alive.

### Why `uniq` needs `sort` first (the mechanism)

`uniq` holds **one line** of state — the previous line — and suppresses the current one only if it is **identical and adjacent**. It never sorts or looks further back. `sort` first guarantees identical lines become adjacent, which is exactly the precondition `uniq` requires. **`sort | uniq -c | sort -rn` is the single most-used analysis pipeline in this entire topic.** Memorize it. (Full demo in the uniq section below.)

---

## grep — In Depth

`grep` = **G**lobal **R**egular **E**xpression **P**rint. It prints lines matching a pattern.

### Core flags

| Flag | Meaning | Why you reach for it |
|---|---|---|
| `-i` | case-insensitive | `grep -i error` catches `Error`, `ERROR`, `error` |
| `-v` | **invert** — print NON-matching lines | Filter OUT noise: `grep -v healthcheck access.log` |
| `-n` | prefix line numbers | Jump straight to the line in your editor |
| `-r` / `-R` | recursive through a directory | `grep -r TODO src/` (`-R` also follows symlinks) |
| `-l` / `-L` | filenames **with** / **without** a match | `grep -rl apiKey .` — which files leak a secret? |
| `-c` | count matching lines (not occurrences) | `grep -c ' 500 ' access.log` |
| `-w` | match **whole word** only | `grep -w id` won't match `uuid` or `idempotent` |
| `-o` | print **only the matched part**, not the line | Extract fields: pull every IP out of a log |
| `-A/-B/-C n` | n lines after / before / both sides of each match | The stack trace *below* an Error line, the request *above* it — ESSENTIAL |
| `-E` | **E**xtended regex (ERE) | `+ ? { } ( ) |` work without backslashes |
| `-F` | **F**ixed string — no regex at all | Fast literal search; `grep -F '1.2.3.4'` treats `.` literally |
| `-q` | quiet — no output, just exit code | For `if` conditions in scripts |
| `-m n` | stop after n matches | `grep -m1` = first hit then quit (fast on huge files) |
| `--include=GLOB` / `--exclude-dir=D` | restrict the file set | `--exclude-dir=node_modules` — ESSENTIAL |

### Regex depth — what the pattern language actually is

```
┌─────────────┬──────────────────────────────────────────────────────────┐
│ ^  $        │ anchors: start of line, end of line                       │
│ .           │ any single character (except newline)                     │
│ [abc]       │ character class: one of a, b, or c                        │
│ [^abc]      │ negated class: any char EXCEPT a, b, c                    │
│ [0-9] [a-z] │ ranges                                                    │
│ [[:digit:]] │ POSIX class (also [:alpha:] [:space:] [:alnum:] [:upper:])│
│ *           │ 0 or more of the preceding    (greedy)                    │
│ +           │ 1 or more                     (ERE; in BRE write \+)      │
│ ?           │ 0 or 1                        (ERE; in BRE write \?)      │
│ {n,m}       │ between n and m               (ERE; in BRE write \{n,m\}) │
│ a|b         │ alternation: a OR b           (ERE; in BRE write a\|b)    │
│ (…)         │ grouping                      (ERE; in BRE write \(…\))   │
│ \.  \*  \/  │ escaping — match a literal dot / star / slash             │
└─────────────┴──────────────────────────────────────────────────────────┘
```

**BRE vs ERE — the backslash trap.** Plain `grep` uses **B**asic **R**egular **E**xpressions, where `+ ? { } ( ) |` are *literal characters* and you must backslash them to get their special meaning. `grep -E` (and the old `egrep`) uses **E**xtended regex, where they are special by default. This is the single most common "why doesn't my regex work" bug.

```bash
grep  'colou\?r' file      # BRE: ? must be escaped to mean "optional"
grep -E 'colou?r' file     # ERE: ? is special without escaping   ← usually clearer
grep  '\(cat\|dog\)' file  # BRE alternation — ugly
grep -E '(cat|dog)' file   # ERE alternation — clean.  PREFER -E.
```

**grep is greedy.** `*`, `+`, `{n,}` all match as much as they can. Basic/extended POSIX grep has **no lazy `*?` operator** (that's a Perl/PCRE feature). If you need lazy matching, use `grep -P` (PCRE, GNU-only, not on Alpine/macOS) or reach for a negated character class instead: to grab up to the *first* quote, use `[^"]*` not `.*`.

### Realistic log-parsing regexes (on an nginx access line)

```bash
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' access.log  # pull every IP address
grep -E  '" 5[0-9]{2} '                access.log  # only 5xx error lines
grep -oE '\[[^]]+\]'                   access.log  # timestamp in brackets
#            ^^^^^ [^]]+ = "everything that isn't a ]" — the anti-greedy trick,
#                  since POSIX grep has no lazy operator
```

**grep family:** `egrep` = `grep -E` (deprecated spelling). `fgrep` = `grep -F`. **`ripgrep` (`rg`)** is a faster modern rewrite (recursive by default, respects `.gitignore`) — great locally, but don't assume it's on the server.

---

## sed — In Depth

`sed` = **s**tream **ed**itor. For each line it loads the text into a buffer (the **pattern space**), runs your script against it, prints the result (unless `-n`), and moves on. It never loads the whole file.

### Substitution — the 90% use case

```
sed 's/old/new/g'
     │ │   │  │
     │ │   │  └── FLAGS: g = every match on the line (default: first only)
     │ │   │            i = case-insensitive   2 = only the 2nd match
     │ │   │            p = print (with -n, print only changed lines)
     │ │   └── replacement text
     │ └── pattern (a regex — BRE by default; use -E for ERE)
     └── s = substitute command
```

```bash
sed 's/localhost/prod-db/'      file   # first "localhost" on each line
sed 's/localhost/prod-db/g'     file   # ALL "localhost" on each line
sed 's/error/ERROR/gi'          file   # global + case-insensitive
sed -n 's/^Host: //p'           file   # print ONLY the value after "Host: "
```

**Different delimiter — essential when the pattern has slashes.** The character right after `s` *is* the delimiter; you can use anything:

```bash
sed 's/\/var\/log/\/data\/log/'  file   # ugh — leaning-toothpick syndrome
sed 's|/var/log|/data/log|'      file   # SAME thing, readable. Use | or # or :
```

### Addresses — which lines to act on

```bash
sed '3d'          file   # delete line 3
sed '2,5d'        file   # delete lines 2 through 5
sed '$d'          file   # delete the LAST line ($ = last)
sed '/^#/d'       file   # delete every line starting with # (regex address)
sed '/^$/d'       file   # delete blank lines
sed '/error/!d'   file   # ! = NEGATE: delete every line that does NOT match /error/
                         #   (i.e. keep only error lines — like grep)
sed -n '10,20p'   file   # -n suppresses auto-print; p prints → show lines 10-20
```

### Other commands, and backreferences (`&`, `\1`)

```bash
sed 'a\ text after'  file             # a=append / i=insert / c=change whole line
sed 'y/abc/xyz/'     file             # transliterate a→x b→y c→z (per-char, like tr)
sed -e 's/a/1/' -e 's/b/2/' file      # multiple expressions with -e
echo "status 500 ok" | sed -E 's/[0-9]{3}/[&]/'       # & = whole match → status [500] ok
echo "Ada Lovelace"  | sed -E 's/(\w+) (\w+)/\2, \1/' # \1 \2 = groups → Lovelace, Ada
```

### `-i` — in-place editing, and THE #1 cross-platform trap

`sed -i` edits the file **on disk** instead of printing to stdout. But GNU sed (Linux) and BSD sed (macOS) disagree on its syntax, and this bites *every* dev who develops on a Mac and deploys to Linux:

```
GNU sed (Linux, Docker):          BSD sed (macOS):
  sed -i 's/a/b/' file            sed -i '' 's/a/b/' file   ← -i REQUIRES an arg
    ↑ -i takes NO argument          ('' = no backup)
  sed -i.bak 's/a/b/' file        sed -i .bak 's/a/b/' file ← space-separated suffix
    ↑ makes file.bak
```

**What actually breaks:** GNU syntax `sed -i 's/a/b/' file` on **macOS** → BSD sed reads `s/a/b/` as the backup *suffix* and `file` as the *script*: `sed: 1: "file": invalid command code f`. BSD syntax `sed -i '' ...` on **Linux** → GNU sed reads `''` as a no-op script and `s/a/b/` as a filename: `sed: can't read s/a/b/`.

**Portable fix:** `sed -i.bak 's/a/b/' file` works on **both** (both read `.bak` as the suffix); delete the `.bak` after. Or avoid `-i` entirely: `sed 's/a/b/' f > f.tmp && mv f.tmp f`.

> Same GNU-vs-BSD divide as topic 01, Mistake 4. Alpine's BusyBox `sed` is a *third* dialect with fewer features still.

---

## awk — In Depth

`awk` is a pattern-scanning language. Its whole model is: **`pattern { action }`**, run once per input line. Omit the pattern and the action runs on every line; omit the action and matching lines are printed (so `awk '/ERROR/'` ≈ `grep ERROR`).

### The automatic variables

```
Input line:   203.0.113.7  GET  /api/orders  500  0.042
              ───┬───────  ─┬─  ─────┬─────   ─┬─  ──┬──
Fields:          $1        $2       $3        $4    $5
              $0 = the ENTIRE line
              NF = 5   (Number of Fields on this line)
              NR = 17  (Number of Records = current line number, across all input)
              FNR      = line number within the CURRENT file (resets per file)
              FS       = input Field Separator  (default: runs of whitespace)
              OFS      = Output Field Separator (default: single space)
```

### Field splitting — the killer feature

```bash
awk '{print $2}'      access.log     # 2nd whitespace-column of every line
awk '{print $NF}'     access.log     # the LAST field ($NF), whatever number it is
awk '{print $(NF-1)}' access.log     # second-to-last
awk -F: '{print $1}'  /etc/passwd    # -F: sets FS to colon → prints usernames
awk -F, '{print $3}'  data.csv       # CSV: -F,   (naive — breaks on quoted commas)
awk 'BEGIN{FS="\t"} {print $1}' f    # tab-separated
```

### Conditions (the "pattern" half)

```bash
awk '$4 == 500'                  access.log   # field 4 is exactly 500
awk '$5 > 1.0 {print $3, $5}'    access.log   # slow requests: response time > 1s
awk '/ERROR/ {print $1}'         app.log      # regex condition + action (≈ grep)
awk 'NR >= 100 && NR <= 200'     file         # line range, like sed 100,200p
awk '$0 ~ /5[0-9][0-9]/'         access.log   # ~ = "matches regex"; NF==0 = blank lines
```

### Arithmetic and SUMMING — what grep and cut simply cannot do

```bash
awk '{sum += $5} END {print sum}' access.log                  # sum column 5
awk '{s += $5; n++} END {printf "avg %.3f\n", s/n}' access.log # average
awk '$5 > max {max = $5} END {print max}' access.log          # max seen
```

### Associative arrays — GROUP BY / counting, the log-analysis workhorse

An awk array is indexed by **strings**. This gives you SQL-style `GROUP BY` in one expression:

```bash
# Count requests per IP (like GROUP BY ip):
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log

# Count each HTTP status, then RANK by piping to sort:
awk '{status[$9]++} END {for (s in status) print status[s], s}' access.log | sort -rn
```

### `printf` and `BEGIN`/`END`

```bash
awk 'BEGIN {print "IP\tCOUNT"}            # BEGIN runs before any input (a header)
     {c[$1]++}
     END {for (ip in c) printf "%-15s %d\n", ip, c[ip]}' access.log
#              └── %-15s = left-justify in 15 cols; %d = integer. Real column formatting.
```

---

## cut — and why it is NOT awk

`cut` slices each line at fixed positions. Fast and simple, with one crippling limitation.

```bash
cut -d: -f1     /etc/passwd     # -d: delimiter is ':', -f1 = field 1 → usernames
cut -d, -f2,4   data.csv        # fields 2 AND 4 from a CSV
cut -d, -f2-    data.csv        # field 2 to the end
cut -c1-8       file            # CHARACTERS 1 through 8 of each line (fixed columns)
```

**The limitation:** `cut -d' '` treats **each single space as its own delimiter**. Commands like `ps` and `ls -l` pad columns with *runs* of spaces, so:

```bash
ps aux | cut -d' ' -f2      # ← BROKEN: returns mostly blanks. Two spaces = an empty field
ps aux | awk '{print $2}'   # ← WORKS: awk's default FS collapses runs of whitespace
```

**Rule:** for clean single-delimiter data (`/etc/passwd`, CSV, TSV), `cut` is perfect and fast. For anything with variable/padded whitespace, use `awk`.

---

## sort — In Depth

```bash
sort           file    # lexical (dictionary) order — the DEFAULT
sort -n        file    # NUMERIC;  -rn = numeric reversed (biggest first)
sort -u        file    # sort + remove duplicates (= sort | uniq)
sort -h        file    # human-numeric: understands 2K, 3M, 1G (for `du -h`)
sort -k2,2 -n  file    # sort by field 2 ONLY, numerically
sort -t: -k3 -n passwd # -t: sets delimiter to ':', sort by field 3 numerically
sort -M        file    # month order: Jan < Feb < … < Dec  (--stable keeps ties' order)
```

**Why lexical sort puts 10 before 2** — the default compares character-by-character as text: `'1' < '2'`, so `"10"` sorts before `"2"` just like `"apple"` before `"b"`. This is *the* classic bug when sorting numbers, byte counts, or response times:

```bash
$ printf '2\n10\n1\n9\n' | sort       $ printf '2\n10\n1\n9\n' | sort -n
1                                      1
10     ← WRONG: 10 before 2            2
2                                      9
9                                      10   ← correct
```

**`sort -h` for `du` output** is a lifesaver — `du -h` prints `4.0K`, `2.1G`, `500M`, and plain `sort -n` can't compare those. `du -h | sort -h` orders them correctly.

---

## uniq — and the #1 gotcha

`uniq` collapses **adjacent** duplicate lines only. **It is useless without `sort` first** (see "How It Works" above).

```bash
sort file | uniq         # deduplicate (or just: sort -u file)
sort file | uniq -c      # PREFIX each unique line with its count
sort file | uniq -d      # print ONLY lines that were duplicated
sort file | uniq -u      # print ONLY lines that appeared exactly once
sort file | uniq -i      # case-insensitive comparison
```

**The canonical "count and rank" pipeline is `sort | uniq -c | sort -rn | head`** — annotated fully in the Exact Syntax Breakdown section below.

---

## tr — translate / squeeze / delete characters

`tr` operates on **single characters** from stdin (it takes no filename — a legit use of `cat` or `<`).

```bash
tr 'a-z' 'A-Z'  < file          # uppercase (per-character translate)
tr '[:upper:]' '[:lower:]' < f  # lowercase, using POSIX classes
tr -d '\r'      < win.txt       # DELETE all carriage returns (fix CRLF → LF)
tr -s ' '       < file          # SQUEEZE runs of spaces into one
tr ',' '\n'     < csvline       # commas → newlines (flatten one CSV row)
```

**The real bug you WILL hit:** a shell script authored on Windows fails with `$'\r': command not found` because every line ends in `\r\n`. bash sees the invisible `\r` as part of the command. Fix (topic 10 diagnosed it with `cat -A`/`file`):

```bash
tr -d '\r' < deploy.sh > deploy.sh.fixed && mv deploy.sh.fixed deploy.sh
```

---

## xargs — turn stdin into command arguments

Many commands (`rm`, `mkdir`, `kill`, `chmod`, `npm`) read their targets from **arguments**, not stdin. `xargs` bridges the gap: it reads items from stdin and lays them out as arguments to a command.

```bash
find . -name '*.tmp' | xargs rm            # delete every .tmp file
echo "1 2 3" | xargs -n1 echo             # -n1 = one arg per invocation → 3 lines
cat urls.txt | xargs -I{} curl -s {}       # -I{} = placeholder; run once per line
find . -name '*.log' | xargs -P4 gzip      # -P4 = 4 parallel processes
```

### Why it exists — and the ARG_MAX limit

The kernel caps the total size of a command's argument list (`ARG_MAX`, ~2 MB; check with `getconf ARG_MAX`). `rm *.log` in a directory with 500,000 files fails with **`Argument list too long`** — that's the *kernel* rejecting the `execve()`, not a bug. `xargs` fixes this by **batching**: it packs as many arguments as fit under the limit, runs the command, then repeats.

### The filename-with-spaces trap — always use `-0` with `find`

Plain `xargs` splits stdin on **whitespace**, so `my report.log` becomes *two* arguments and you delete the wrong things. Fix: `find -print0` emits NUL (`\0`) between names (the only byte illegal in a filename) and `xargs -0` splits on NUL:

```bash
find . -name '*.log' -print0 | xargs -0 rm      # SAFE with any filename
find . -name '*.log' | xargs rm                 # BROKEN if any name has a space
```

Also: `xargs -r` (`--no-run-if-empty`) skips the command entirely on empty stdin. And for pure deletion, `find … -delete` needs no `xargs` and is safest — use `xargs` when the action isn't a `find` primitive (`gzip`, `curl`, `git add`).

---

## Also in the Toolbox

```bash
grep ERROR app.log | head -20            # first 20 matches (grep stops early)
somecmd | tee out.log | grep ERROR       # tee = write to file AND pass on (topic 12)
comm -12 <(sort a) <(sort b)             # lines common to both sorted files
paste f1 f2                              # glue files side-by-side, tab-separated
join -t, -1 1 -2 1 orders.csv users.csv  # SQL-style join on a key column
```

### jq — because a Node dev's logs are JSON

Modern Node apps (pino, winston, bunyan) emit **one JSON object per line**. `grep`/`awk` see them as dumb text; `jq` parses them properly:

```bash
# Given lines like: {"level":"error","msg":"db timeout","reqId":"a1","ms":42}
jq -r 'select(.level=="error") | .msg' app.log   # every error message, unquoted
jq -rs 'map(.ms) | add/length'         app.log   # average latency
#     │└── s = slurp all lines into one array;  add/length = mean
#     └── r = raw output (no JSON quotes)
```

`jq` isn't always preinstalled (`apt install jq`), but for JSON logs it beats every regex you could write.

---

## Exact Syntax Breakdown

The canonical count-and-rank pipeline, annotated end to end:

```
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
│    │            │            │      │    │    │    │     │
│    │            │            │      │    │    │    │     └── keep the first 10
│    │            │            │      │    │    │    └── -n numeric (not lexical)
│    │            │            │      │    │    └── -r reverse (biggest first)
│    │            │            │      │    └── -c prefix each line with its count
│    │            │            │      └── uniq collapses ADJACENT dupes only
│    │            │            └── sort makes identical lines adjacent (uniq's precondition)
│    │            └── the file to read
│    └── action: print field 1 (the IP) of every line
└── awk with no pattern → action runs on every line
```

```
grep -oE '" 5[0-9]{2} ' access.log
│    ││   │              │
│    ││   │              └── the file to search
│    ││   └── ERE pattern: a quote, space, 5, two digits, space  → 5xx status codes
│    │└── -E = Extended regex, so {2} works without backslashes
│    └── -o = print ONLY the matched text, not the whole line
└── grep — Global Regular Expression Print
```

---

## Example 1 — Basic

```bash
# A tiny "log" to practice on:
printf 'GET /a 200\nGET /b 500\nGET /a 500\nGET /a 200\nGET /c 500\n' > /tmp/mini.log

# grep — filter to just the errors:
grep 500 /tmp/mini.log            # the three lines ending in 500

# grep -c — count them:
grep -c 500 /tmp/mini.log         # 3

# awk — pull one column (the path is field 2):
awk '{print $2}' /tmp/mini.log    # /a /b /a /a /c

# awk condition — only 500s, print the path:
awk '$3 == 500 {print $2}' /tmp/mini.log   # /b /a /c

# The workhorse: count requests per path, ranked:
awk '{print $2}' /tmp/mini.log | sort | uniq -c | sort -rn
#    3 /a
#    1 /b
#    1 /c

# sed — rewrite the method in a stream (doesn't touch the file):
sed 's/GET/POST/' /tmp/mini.log   # every GET → POST on stdout

# tr — uppercase the whole thing:
tr 'a-z' 'A-Z' < /tmp/mini.log    # GET /A 200 ...
```

---

## Example 2 — Production Scenario

**Situation:** 03:00. Your Node API behind nginx is slow and throwing errors. You have `/var/log/nginx/access.log`. You're going to answer five questions in five one-liners. First, **know your log format** (nginx `combined`):

```
203.0.113.7 - - [12/Jul/2026:02:14:31 +0000] "GET /api/orders HTTP/1.1" 500 1234 "-" "curl/8.1" 0.042
─────┬─────         ────────────┬──────────  ──┬─  ────────┬──────────  ─┬─ ──┬─           ──┬──  ──┬──
    $1 IP                    [$4 $5] time       │      $7 endpoint       │  bytes         agent  response
                                            $6 method   ($8=proto)      $9 status                  time ($NF)
```
(Field numbers are for awk's default whitespace split. `$9` is the status code; `$NF` — the last field — is the request time, since we added it via `log_format`.)

### Build it up, one stage at a time

**Q1 — Top 10 IPs by request count** (who is hammering us?):
```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
#    └ pull the IP      └ group    └ count   └ rank    └ top 10
```
```
   48213 203.0.113.7        ← one IP made 48k requests — likely the attacker/runaway client
    9102 198.51.100.44
    3341 203.0.113.9
     ...
```

**Q2 — How many 5xx responses total?**
```bash
awk '$9 ~ /^5/' access.log | wc -l
#    └ keep lines whose status field starts with 5, then count them
# 14028
```

**Q3 — Which endpoints return the most 500s?** (where is the bug?):
```bash
awk '$9 == 500 {print $7}' access.log | sort | uniq -c | sort -rn | head
#    └ only 500s, emit the endpoint  └────── count & rank ──────┘
```
```
   13990 /api/orders        ← the smoking gun: /api/orders is the broken endpoint
      21 /api/users
      17 /api/health
```

**Q4 — Requests per minute during the incident window** (when did it spike?):
```bash
grep '12/Jul/2026:02:1' access.log | awk '{print substr($4,2,17)}' | sort | uniq -c
#     └ narrow to the window          └ substr chops [timestamp to minute precision
#   1198 12/Jul/2026:02:11
#   8842 12/Jul/2026:02:14      ← 7x normal — the spike started at 02:14
```

**Q5 — Average response time, and the slowest endpoint** (awk does math grep can't):
```bash
# Overall average request time (last field):
awk '{sum += $NF; n++} END {printf "avg %.3fs over %d reqs\n", sum/n, n}' access.log
# avg 0.087s over 63019 reqs

# Average response time PER endpoint, slowest first:
awk '{sum[$7] += $NF; cnt[$7]++}
     END {for (e in sum) printf "%.3f  %s\n", sum[e]/cnt[e], e}' access.log \
  | sort -rn | head
# 2.412  /api/orders      ← /api/orders isn't just erroring, it's also the slowest
# 0.041  /api/users
# 0.009  /api/health
```

**What you concluded in 90 seconds, no spreadsheet:** IP `203.0.113.7` is flooding you; the damage is concentrated in `/api/orders`, which is both throwing 13,990 of your 14,028 500s *and* averaging 2.4s; and it all started at 02:14. That's the entire triage — done with `awk`, `sort`, `uniq`, `grep`.

---

## Common Mistakes

### Mistake 1 — `uniq` without `sort`

**Wrong:** `awk '{print $1}' access.log | uniq -c`

**Root cause:** `uniq` compares each line only to the **immediately preceding** one. IPs are scattered through the log, so identical ones are rarely adjacent and `uniq` counts each little run separately — you report "203.0.113.7 appeared 3 times" when it appeared 48,000.

**Right:** `awk '{print $1}' access.log | sort | uniq -c | sort -rn`
**Prevention:** `uniq` and `sort` travel together, always. No counts needed? `sort -u` does both.

---

### Mistake 2 — lexical sort on numbers

**Wrong:** `du -s * | sort -r | head` to find the biggest directory.

**Root cause:** without `-n`, `sort` compares byte-counts as **text**: `"9000"` sorts *above* `"85000000"` because `'9' > '8'` at the first character. You pick the wrong "biggest" directory.

```bash
du -s * | sort -rn | head       # RIGHT — numeric
du -h * | sort -rh | head       # RIGHT for human units (4.0K, 2.1G) — note -h
```
**Prevention:** numeric key → `-n`; `du -h`-style human sizes → `-h`.

---

### Mistake 3 — GNU `sed -i` syntax on macOS (and vice-versa)

**Wrong (repo script run on a Mac):** `sed -i 's/DEBUG/INFO/' config.js`

**Root cause:** BSD sed (macOS) requires an argument after `-i`; with none it reads `s/DEBUG/INFO/` as the backup suffix and `config.js` as the *script* → `sed: 1: "config.js": invalid command code c`. (BSD syntax on Linux fails the mirror-image way.)

**Right (portable):** `sed -i.bak 's/DEBUG/INFO/' config.js && rm config.js.bak` — works on GNU **and** BSD. Or the tmp-file pattern: `sed '…' f > f.tmp && mv f.tmp f`.
**Prevention:** never hardcode bare `-i` in a shared script. (Same root cause as topic 01, Mistake 4.)

---

### Mistake 4 — `cut -d' '` on padded columns

**Wrong:** `ps aux | grep node | cut -d' ' -f2` to get the PID.

**Root cause:** `ps` pads columns with **runs** of spaces; `cut -d' '` treats each single space as a boundary, so a run of 3 spaces makes 2 empty fields. Field 2 is usually one of those empties → blank output, "randomly" failing script.

**Right:** `ps aux | grep '[n]ode' | awk '{print $2}'` — awk collapses whitespace runs (and `[n]ode` stops grep matching itself).
**Prevention:** `cut` for clean delimiters (`:`, `,`); `awk` for whitespace-formatted command output.

---

### Mistake 5 — `find | xargs` with spaces in filenames

**Wrong:** `find /uploads -name '*.tmp' | xargs rm`

**Root cause:** `xargs` splits stdin on whitespace, so `vacation photo.tmp` becomes two arguments — `rm` errors, or deletes a *different* file named `vacation`. With user-supplied upload filenames this is real data loss.

**Right:** `find /uploads -name '*.tmp' -print0 | xargs -0 rm` (NUL-separated), or simply `find … -delete`.
**Prevention:** any `find | xargs` touching real files gets `-print0`/`-0`; prefer `-delete` for deletion.

---

## Hands-On Proof

```bash
# PROVE IT: grep's exit code drives `if`. (0 = found, 1 = not found)
grep -q root /etc/passwd && echo "found"      # prints found
grep -q zzzz /etc/passwd && echo "found" || echo "missing"   # prints missing
echo "last exit code: $?"

# PROVE IT: uniq only sees ADJACENT duplicates.
printf 'a\nb\na\nb\na\n' | uniq -c            # five lines of "1 x" — nothing collapsed
printf 'a\nb\na\nb\na\n' | sort | uniq -c     # "3 a" and "2 b" — correct

# PROVE IT: lexical vs numeric sort.
printf '2\n10\n1\n' | sort        # 1, 10, 2   (text order)
printf '2\n10\n1\n' | sort -n     # 1, 2, 10   (numeric)

# PROVE IT: awk sums (grep/cut can't) and its array is a GROUP BY.
seq 100 | awk '{s+=$1} END{print s}'                            # 5050
printf 'a\nb\na\nc\na\n' | awk '{c[$0]++} END{for(k in c) print c[k],k}' | sort -rn

# PROVE IT: cut breaks on padded whitespace; awk doesn't.
ps aux | head -3 | cut -d' ' -f2       # mostly blanks
ps aux | head -3 | awk '{print $2}'    # real PIDs

# PROVE IT: BRE vs ERE — same pattern, different results.
echo "color colour" | grep -oE 'colou?r'      # matches both (ERE, bare ?)
echo "color colour" | grep  -o 'colou?r'      # matches NOTHING (literal '?' in BRE)

# PROVE IT: xargs batches around the kernel's ARG_MAX limit.
getconf ARG_MAX                        # ~2097152 bytes
seq 1 100000 | xargs echo | wc -l      # >1 line: xargs ran echo in several batches

# PROVE IT: -print0 / -0 survive spaces.
mkdir -p /tmp/sp && touch "/tmp/sp/a b.txt"
find /tmp/sp -name '*.txt' -print0 | xargs -0 ls -l    # lists "a b.txt" correctly
```

---

## Practice Exercises

### Exercise 1 — Easy

```bash
printf 'apple\nbanana\napple\ncherry\nbanana\napple\n' > /tmp/fruit.txt
```
In the terminal:
1. Count how many times each fruit appears, ranked most-frequent first.
2. Print only the fruits that appear more than once.
3. Uppercase the whole file with `tr`.
4. Use `grep -c` to count lines containing `an`.
5. Use `awk` to print each line prefixed with its line number (don't use `-n`; use `NR`).

### Exercise 2 — Medium

```bash
# A fake CSV of orders: id,user,amount,status
cat > /tmp/orders.csv <<'EOF'
1,alice,42.00,paid
2,bob,13.50,failed
3,alice,99.99,paid
4,carol,5.00,paid
5,bob,7.25,failed
6,alice,20.00,refunded
EOF
```
Answer each with a one-liner (mind the CSV header if you add one):
1. Total revenue from `paid` orders only (sum column 3 where column 4 == "paid"). Use `awk -F,`.
2. Number of orders per user, ranked.
3. The user who spent the most across all `paid` orders (hint: awk associative array keyed by user + `sort -rn`).
4. Convert the file to tab-separated using `tr` (or `sed`).

### Exercise 3 — Hard (Production Simulation)

```bash
# Generate a realistic nginx access log:
awk 'BEGIN{
  srand(); ips[0]="203.0.113.7"; ips[1]="198.51.100.4"; ips[2]="203.0.113.9";
  ep[0]="/api/orders"; ep[1]="/api/users"; ep[2]="/api/health";
  for(i=0;i<20000;i++){
    ip=ips[int(rand()*3)]; e=ep[int(rand()*3)];
    st=(e=="/api/orders" && rand()<0.6)?500:200;
    rt=(st==500)? 1.5+rand() : rand()*0.1;
    printf "%s - - [12/Jul/2026:02:%02d:%02d +0000] \"GET %s HTTP/1.1\" %d 512 \"-\" \"curl\" %.3f\n",
           ip, int(rand()*60), int(rand()*60), e, st, rt;
  }
}' > /tmp/access.log
```
Using only the tools from this topic (no editor, no spreadsheet), answer:
1. Top 3 IPs by request count.
2. Total number of 5xx responses.
3. The endpoint responsible for the most 500s, and how many.
4. Average response time for `/api/orders` vs `/api/health` (two numbers). Which is slower and by how much?
5. Requests per minute, to find the busiest minute.
6. Build ONE pipeline that prints: for each endpoint, its request count, its 500 count, and its average response time — sorted by 500 count descending. (Hint: a single `awk` with three associative arrays and an `END` loop, piped to `sort`.)

Paste every command and its output.

---

## Mental Model Checkpoint

1. **Why is `sort | uniq -c | sort -rn` the workhorse pipeline, and what does each of the three stages contribute?**
2. **`grep -E '(cat|dog)'` works but `grep '(cat|dog)'` matches nothing. Why? What is BRE vs ERE?**
3. **You have `ps aux | grep node`. Why does `cut -d' ' -f2` fail to get the PID but `awk '{print $2}'` succeeds?**
4. **What is the one thing `awk` can do that `grep` and `cut` fundamentally cannot? Give an example.**
5. **On your Mac, `sed -i 's/a/b/' file` errors, but the identical command works on the Linux server. Why, and what's the portable fix?**
6. **What does `find . | xargs rm` do to a file named `my notes.txt`, and what is the correct incantation?**
7. **In `awk`, what are `$0`, `$1`, `$NF`, `NR`, and `FS`?**
8. **Why can you `grep pattern hugefile | head` on a 40 GB file and get an answer instantly, but `sort hugefile | head` has to churn?**

---

## Quick Reference Card

| Command | What It Does | Key Flags |
|---------|-------------|-----------|
| `grep` | Print lines matching a regex | `-i` ignore-case, `-v` invert, `-n` line#, `-r` recursive, `-l` filenames, `-c` count, `-w` word, `-o` only-match, `-A/-B/-C n` context, `-E` ERE, `-F` fixed, `-q` quiet, `-m n` max, `--exclude-dir` |
| `sed` | Stream editor (transform lines) | `s/a/b/g` sub, `-n …p` print, `Nd`/`/re/d` delete, `-E` ERE, `-i.bak` in-place (portable), `-e` multi-expr, `&`/`\1` backrefs |
| `awk` | Field-aware scripting per line | `-F:` set FS, `$1..$NF`, `NR`/`NF`, `BEGIN`/`END`, `{sum+=$3}`, `arr[$1]++`, `printf` |
| `cut` | Slice fixed fields/columns | `-d: -f1`, `-f2-`, `-c1-8`. Fails on padded whitespace |
| `sort` | Order lines | `-n` numeric, `-r` reverse, `-k2,2` key, `-t:` delim, `-u` unique, `-h` human sizes, `-M` month |
| `uniq` | Collapse ADJACENT dupes | `-c` count, `-d` dupes only, `-u` uniques only, `-i` ignore-case. **Needs `sort` first** |
| `tr` | Translate/delete/squeeze chars | `'a-z' 'A-Z'`, `-d '\r'`, `-s ' '`, `[:upper:]` classes. stdin only |
| `xargs` | stdin → command arguments | `-n1`, `-I{}`, `-P4` parallel, `-0` NUL-safe, `-r` no-run-if-empty |
| `jq` | Parse/filter JSON logs | `-r` raw, `-s` slurp, `select(.level=="error")`, `.field` |
| `tee` | Write to file AND pass on | `-a` append (topic 12) |
| `wc` | Count lines/words/bytes | `-l`, `-w`, `-c` (topic 10) |

---

## When Would I Use This at Work?

### Scenario 1: Which endpoint is throwing 500s?
Your API alerts on error rate. You SSH in and, in one line, find the culprit:
```bash
awk '$9 == 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```
The top line is your broken route. No dashboard, no export — 200ms to answer.

### Scenario 2: Tidy a config across environments
You're switching a service from a hardcoded host to an env var across a repo:
```bash
grep -rl 'http://localhost:5432' src/ \
  | xargs sed -i.bak 's|http://localhost:5432|process.env.DB_URL|g'
# grep -rl finds the files; xargs feeds them to sed; -i.bak edits in place, portably
find src -name '*.bak' -delete       # clean up backups when you've verified the diff
```

### Scenario 3: Parse structured JSON logs from a pino/winston app
Your Node app logs one JSON object per line. Find the slowest error requests:
```bash
jq -rs 'map(select(.level=="error" and .ms>500)) | sort_by(.ms) | reverse | .[:5][]
        | "\(.ms)ms  \(.reqId)  \(.msg)"' /var/log/app/api.log
# The 5 slowest error requests, with their request IDs to trace end-to-end.
```

---

## Connected Topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Builds on** | 10 — Viewing and Reading Files | You found the file with `less`/`tail`/`zgrep`; now these tools extract meaning from it |
| **Builds on** | 04 — Inodes / 05 — Permissions | `grep -r` walks directory entries; "Permission denied" while grepping is the kernel checking mode bits |
| **Deeply uses** | 12 — Pipes and Redirection | Every `|`, `>`, `2>&1` here is topic 12. The pipeline IS the composition model |
| **Next** | 12 — Pipes and Redirection | Formalizes stdin/stdout/stderr — the contract all these tools share |
| **Used by** | 13 — Shell Scripting | `grep -q` in `if`, `awk` for computed values, exit codes ($?) drive script logic |
| **Used by** | 17 — Process Management | `ps aux \| grep \| awk '{print $2}' \| xargs kill` — the classic "kill by name" chain |
| **Used by** | 23 — Logs and Log Management | This entire toolkit is how you read `/var/log` and `journalctl` output under pressure |
| **Used by** | 28 — Ports and Sockets / 30 — curl | `ss -tlnp \| grep :3000`, parsing `curl -v` headers, filtering `netstat` — all text processing |
