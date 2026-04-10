# HuMaCli — implementation prompt

Use this prompt to reproduce or extend HuMaCli with any capable language model.

---

```
You are building HuMaCli — a standalone offline natural language to Mac CLI
command translator. Single HTML file, no dependencies, no network calls,
no LLM at runtime. User types natural language, gets a shell command to
review and run themselves.

## architecture — 3-layer pipeline

### layer 1 — intent resolution
Single pass over all tokens simultaneously. Produces three 32-bit integers:
- verbBit   — which verb/intent was found
- nounBit   — which noun/subject was found
- majorBit  — which major modifier was found (the discriminator that
              switches command family)

verb + noun alone is often ambiguous:
  list + file → ls? find? grep?  — cannot know without major modifier
  kill + process → kill? pkill?  — cannot know without pid vs name

The major modifier bit resolves the ambiguity. It is not a minor adjustment
— it is a family switch. Without it the system would have to guess.
The dispatch key must be three integers, not two.

verb + noun + major_modifier together → dispatch key → command family.
Do NOT determine family from verb alone. Do NOT determine family from
first word alone.

Input is a bag of words for single-word tokens — order does not matter
for them. Multi-word constructs (larger than 1MB, modified in last 7 days,
my Downloads) use local adjacency. The output integers are order-independent
even though extraction uses adjacency for multi-word phrases.

### layer 2 — slot extraction
Receives confirmed family from layer 1. Extracts only the slots relevant
to that family. No cross-contamination between families.

find family needs:  path, glob, size, mtime, recursive, depth, iname
grep family needs:  term, path, glob, iname, recursive
ps   family needs:  term, lines, sort, tree
kill family needs:  signal, pid OR appname
tail family needs:  path, lines, follow
pip  family needs:  pkg, action

Family must be confirmed before extraction runs. Context-dependent tokens
(python, force) mean different things in different family contexts.
Layer 2 errors are always layer ordering violations.

### layer 3 — normalise + fill
Applies minor flags, platform detection, normalises size units for
BSD/GNU find, substitutes slot values into template.

Mac BSD find uses 512-byte blocks: +1M → +2048, +1G → +2097152, +1k → +2

## vocabulary — three closed sets

### structural vocabulary (governs skeleton)

verbs (16):
  list, find, create, delete, read, copy, move, navigate, grep, tail,
  date, inspect, kill, open, source, pip

nouns (29):
  file, directory, process, disk, app, log, package, venv, script,
  requirements, symlink, archive, image, video, audio, document, config,
  database, network, user, permission, git, remote, container, service,
  env_var, alias, history, clipboard

major modifiers (10 — only discriminators that switch family):
  glob, size, modified, containing, pid, follow, app, lines, sort, tree

Minor flags (recursive, force, hidden, iname, empty, count) are NOT major
modifiers — they do not switch family.

### value vocabulary (open-ended)
Paths, file patterns, process names, package names, search terms, numbers
are NOT in any vocabulary map. They are extracted positionally from what
surrounds structural tokens. Nothing useful is discarded.

Parse failure happens only when the structural skeleton cannot be resolved
— when verb+noun+major has no dispatch entry. Unknown values never cause
parse failure.

## tokeniser rules
- preserve original case for path/pkg/appname values
- use lowercase only for map lookups
- strip surrounding quotes from tokens ("*.js" → *.js)
- bracket notation: file(*.log) process(python) size(>1GB)
  name(*.js) containing(ERROR) modified(<7days) lines(50) top(5)
- bracket takes priority over inferred value
- path brackets NOT allowed — paths come from natural language only

## token role assignment — strict priority order
1. bracket tokens — noun(filter) or modifier(value)
2. my-expansion — my home→~ / my Downloads→~/Downloads
3. path tokens — bare ~/... /... and dest prepositions to/into
4. verb — longest match first
5. noun — longest match, skip already-consumed tokens
6. major modifiers — lang→glob, bare extension, size phrase, mtime,
   containing, app name, follow, sort, top N, N lines, pid, tree
7. minor flags — recursive, force, hidden, iname, empty, count
8. family-specific — signal/pid only in kill context, pkg in pip context,
   unknown word in kill context = process name

## special rules
- lang tokens (python→*.py) in process context → containing, not glob
- file+directory+glob → if directory explicit: find|dirname intent
- process noun + list verb → upgrade to inspect verb
- open + app modifier → noun = app
- kill + unknown word → process name (pkill target)
- signal extracted only in kill context
- force in kill context → signal -9, not rm -f
- top N → lines modifier

## path normalisation
home/~ → ~, here/current/. → ., parent/.. → .., root → /

## my-expansion
my home [folder] → ~
my <word> folder/dir → ~/word (preserve original case)
my downloads/documents/desktop/dev/src/code/work → ~/word

## dispatch table
key = verbBit:nounBit:majorBit
one verb bit per entry — no ORed verb keys
partial match fallback: highest popcount wins

## command families and templates
ls:           ls -lh {path}
find:         find {path} -type f -name "{glob}" -size {size} -mtime {mtime} 2>/dev/null
grep:         grep -r "{term}" {path}
find|grep:    find {path} -type f -name "{glob}" 2>/dev/null | xargs grep "{term}"
find|dirname: find {path} -type f -name "{glob}" 2>/dev/null | xargs -I% dirname % | sort -u
ps:           ps aux | grep "{term}"  /  ps aux -r | head -n {lines}
kill:         kill {signal} {pid}
pkill:        pkill -i {appname}
tail:         tail -f {path}  /  tail -n {lines} {path}
cp/mv/rm/mkdir/touch/cd/cat: standard forms
open:         open -a "{appname}" (Mac) / xdg-open (Linux)
source:       source {path}/venv/bin/activate
pip:          python3 -m pip install {pkg}
date:         date "+%Y-%m-%d %H:%M:%S"

## platform handling
detect via navigator.userAgent — isMac
open app:    open -a "Name" (Mac) / xdg-open (Linux)
ps sort:     ps aux -r (Mac) / ps aux --sort=-%cpu (Linux)
find size:   Mac BSD find uses 512-byte blocks
             +1M → +2048, +1G → +2097152, +1k → +2

## display format
verb      : 0x00000001  (list)
noun      : 0x00000001  (file)
major     : 0x0000000d  (glob(*.py), size(+2048), path(~/Downloads))
key       : 0x00000001:0x00000001:0x0000000d
family    : find
predicate : list(file, path(~/Downloads), glob(*.py), size(+2048))
command   : find ~/Downloads -type f -name "*.py" -size +2048 2>/dev/null

## section markers for sed-addressable vocabulary extension
//section:name  — vocabulary or logic block
//func:name     — function definition
//rule:name     — rule inside function

## error output
unable to parse — verb(X) noun(Y) major(Z) — no dispatch match
Always produce best-guess command. Never refuse.
User decides whether to run it.

## examples that must work
list python files in my Downloads larger than 1MB
find file(*.log) in /var/log modified in last 7 days containing ERROR
show hidden files in current directory
find .py files in current folder recursively
what folders under my home have python files
list process(python)
list top 5 processes
list processes by cpu
kill process 1234
force kill firefox
kill node
open chrome
copy ~/docs/report.pdf to ~/backup
delete folder /tmp/cache forcefully
tail follow /var/log/syslog
show last 50 lines of ~/logs/app.log
find files in ~/humaipam name(*.js)
activate venv in ~/myproject
install requests
find ~/src -name "*.js" | grep TODO
```

---

*© 2026 Yaroslav Vlasov / HuMaDev*
*github.com/dothuma/humacli*
