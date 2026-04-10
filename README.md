# HuMaCli

**Human Machine Cli** — natural language to Mac command line translator.

Type what you want to do in plain English. Get a shell command back.
Copy it. Paste it into your terminal. Run it at your own judgment.

No installation. No API key. No network call. One HTML file.

---

## what it does

```
you type:    list python files in my Downloads larger than 1MB
you get:     find ~/Downloads -type f -name "*.py" -size +2048 2>/dev/null

you type:    force kill firefox
you get:     pkill -i -9 firefox

you type:    find log files modified in last 7 days containing ERROR
you get:     find . -type f -mtime -7 2>/dev/null | xargs grep "ERROR"

you type:    activate venv in ~/myproject
you get:     source ~/myproject/venv/bin/activate
```

HuMaCli is a translator only. It does not execute commands.
You decide what runs on your machine.

---

## how to run on Mac

**option 1 — download and open**

1. Download `index.html` from this repository
2. Double-click the file — it opens in your default browser
3. Type a command in natural language
4. Copy the result and paste into Terminal

**option 2 — clone and open**

```bash
git clone https://github.com/dothuma/humacli.git
cd humacli
open index.html
```

That is it. No `npm install`. No build step. No server.

**tested on:** Mac with Chrome. Should work on Safari and Firefox.
Should work on Linux — `open` commands will use `xdg-open` automatically.

---

## syntax

Natural language works for most commands:

```
list python files in my Downloads larger than 1MB
find .js files in ~/src recursively
show hidden files in current directory
kill process 1234
tail follow /var/log/syslog
show last 50 lines of ~/logs/app.log
install requests
activate venv in ~/myproject
copy ~/docs/report.pdf to ~/backup
delete folder /tmp/cache forcefully
open chrome
list top 5 processes
list processes by cpu
what folders under my home have python files
```

**bracket notation** for explicit typing:

```
file(*.log)          — file with glob filter
process(python)      — process with grep filter
size(>1GB)           — size comparison
containing(ERROR)    — content search term
name(*.js)           — filename pattern
modified(<7days)     — time filter
lines(50)            — line count
top(5)               — top N processes
```

**pipe** for compound commands:

```
find ~/src -name "*.js" | grep TODO
```

---

## supported commands

| intent | examples |
|--------|---------|
| list / find files | `list`, `find`, `show`, `search`, `what`, `where` |
| grep content | `containing`, `matching`, `search inside` |
| process management | `ps`, `kill`, `pkill`, `top` |
| file operations | `copy`, `move`, `delete`, `create`, `read` |
| navigation | `cd`, `go to` |
| tail / monitor | `tail`, `follow`, `watch` |
| python environment | `activate venv`, `install`, `pip` |
| open apps | `open chrome`, `launch finder` |
| date / time | `show date`, `current time` |

---

## path shortcuts

```
my Downloads     → ~/Downloads
my home          → ~
my projects      → ~/projects
current          → .
here             → .
home             → ~
```

---

## disclaimer

HuMaCli generates shell commands from natural language. You are responsible
for reviewing and executing them. Commands involving `rm`, `kill`, or system
paths can be destructive. Always read before you run.

This software is provided as-is, without warranty of any kind.
See [LICENSE](LICENSE) for details.

---

## how it works

HuMaCli is a deterministic three-layer transformer:

1. **layer 1** — scans all tokens simultaneously, maps verb + noun +
   major modifier to a 96-bit dispatch key
2. **layer 2** — extracts slots for the confirmed command family
3. **layer 3** — fills a command template with normalised values

Read [TRANSFORMER.md](TRANSFORMER.md) for a full technical explanation
including how the architecture maps onto LLM transformer layers.

---

## extending the vocabulary

Every vocabulary section is marked with `//section:name` comments.
Extract a section with `sed`, extend it with an AI, replace it:

```bash
# extract verbs section
sed -n '/\/\/section:verbs/,/\/\/section:nouns/p' index.html

# replace section
sed -i '/\/\/section:verbs/,/\/\/section:nouns/{
  /\/\/section:verbs/r updated-verbs.js
  /\/\/section:nouns/!d
}' index.html
```

---

## built with

[dothuma](https://humadev.ai) — human machine collaborative development methodology.

---

## license

Apache 2.0 — see [LICENSE](LICENSE)

*© 2026 Yaroslav Vlasov / HuMaDev*
