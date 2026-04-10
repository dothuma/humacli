% fqnp: humacli/md/HUMACLI_TRANSFORMER_0410.md
# HuMaCli — human language to Mac CLI deterministic transformer

*three transformation layers turn limited vocabulary human language
into a shell command, and why this maps directly on how LLM machine
transformers do*

*Fri Apr 10 10:03:49 EDT 2026*

---

## why this exists

The command line is legacy infrastructure. It has been around since the
1970s and it is not going anywhere — every serious Mac and Linux workflow
eventually touches a terminal. But for newcomers to CLI, the gap between
knowing what you want to do and knowing how to express it as a shell command
is a real barrier. You know you want to find large python files in your
Downloads folder. You do not know that the right incantation is
`find ~/Downloads -type f -name "*.py" -size +2048 2>/dev/null`.

HuMaCli — Human Machine Cli — is proposed as a save harbour for that gap.
It is an interface layer between human language and the Mac command line.
The name is intentional: **Hu** for human, **Ma** for machine, **Cli** for
command line interface. It also echoes the dothuma methodology — human and
machine working together, each doing what it does best.

The human states intent in natural language. HuMaCli translates it to a
shell command. The human reviews the command and decides whether to run it.

This document explains how that translation works — not as a black box, but
as a transparent three-layer transformation pipeline that maps directly onto
how large language model transformers operate internally.

---

## important — scope and responsibility

HuMaCli is a **translator only**. It does not execute commands.

The implementation is a single standalone `index.html` file — open it in
any browser on your Mac, type natural language, get a shell command. Copy
the command. Check it. Paste it into your terminal. Run it (at your own risk:).
Do not run any rm or kill command if you are not absolutely sure.

The execution layer — reading from a node.js process, piping to the shell,
capturing output — is deliberately omitted. That part is straightforward
engineering and adding it would change the character of the tool from a
playground into a system with real consequences. The translation layer is
the interesting part. That is what this document is about.

**You execute the resulting command at your own risk.** HuMaCli makes no
guarantees about the correctness, safety, or appropriateness of generated
commands for your specific system and context. We provide this as an
interesting subject for exploration, without obligations of any kind.
Always read a command before running it, especially anything involving
`rm`, `kill`, or operations on system paths.

---

## what HuMaCli is

HuMaCli is a deterministic transformer. Given the same input it always
produces the same output. No probability, no sampling, no network call,
no API key. Open the HTML file, type, get a command.

It operates on a **closed structural vocabulary**:

```
16 verbs      list, find, create, delete, read, copy, move, navigate,
              grep, tail, date, inspect, kill, open, source, pip

29 nouns      file, directory, process, disk, app, log, package, venv,
              script, requirements, symlink, archive, image, video, audio,
              document, config, database, network, user, permission, git,
              remote, container, service, env_var, alias, history, clipboard

10 major      glob, size, modified, containing, pid, follow,
modifiers     app, lines, sort, tree
```

These 55 structural terms define the skeleton of every command — the intent,
the subject, the discriminator that selects which command to use.

Everything else — paths, file patterns, process names, package names, search
terms, numbers — is **open-ended value vocabulary**. `~/Downloads`,
`node`, `requests`, `ERROR`, `*.js`, `1234` are not in any map. They are
extracted positionally from what surrounds the structural tokens.

The closed vocabulary governs structure. The open values fill the slots
that structure creates. Nothing useful is discarded.

---

## the input

```
list python files in my Downloads larger than 1MB
```

Ten words. Intent, subject, path, type filter, size constraint — all
mixed together with no explicit labels.

The job is to produce:

```
find ~/Downloads -type f -name "*.py" -size +2048 2>/dev/null
```

Three transformation layers run between those two strings.

---

## the token

Before any layer runs, the input is split into tokens:

```
[list] [python] [files] [in] [my] [Downloads] [larger] [than] [1MB]
```

Each token carries two representations simultaneously:

```javascript
{
  pos:      5,           // position in original string
  word:     'Downloads', // original case — preserved for output values
  lower:    'downloads', // lowercase — used for map lookups only
  role:     null,        // assigned by layer 1 — or stays null
  value:    null,        // extracted value once role assigned
  consumed: false        // true once claimed by a layer
}
```

Original case is preserved because paths and names in the final command
must appear exactly as the user typed them. Lowercase is used only for
matching. Every token goes through layer 1 and gets classified:

- **structural role** — matched to a vocabulary map: verb, noun, major modifier,
  preposition, path expansion trigger
- **value role** — not in any map but carrying a value: path token, bare
  extension, number, unknown word in kill context
- **null** — genuinely unrecognised noise, silently ignored

Nothing that carries meaning is discarded.

---

## layer 1 — intent resolution

### the problem: verb + noun is often ambiguous

This is the central architectural insight of HuMaCli.

Verb and noun together are not enough to determine which command to use:

```
list + file    → ls?  find?  grep?    — ambiguous
list + process → ps?  top?            — unambiguous
kill + process → kill <pid>?  pkill?  — ambiguous
tail + file    → tail -f?  tail -n N? — ambiguous
```

Three of those four combinations cannot be resolved without more information.
The dispatch key cannot be two integers — it must be three. The third integer
is the **major modifier bit** — the discriminator that resolves the ambiguity
and maps to exactly one command:

```
list + file + glob        → find -name
list + file + containing  → grep
list + file + size        → find -size
list + file + nothing     → ls
kill + process + pid      → kill <pid>
kill + process + app      → pkill <name>
tail + file + follow      → tail -f
tail + file + lines       → tail -n N
```

The major modifier bit is not a minor adjustment. It is a family switch.
Without it, the system would have to guess. With it, every verb+noun+major
combination maps to exactly one command family.

This is also exactly where LLMs fail when they get CLI translation wrong.
They see `list` and `file` and default to `ls` without weighting the
`containing` or `size` token strongly enough to switch families. The major
modifier is underattended and the wrong command is generated.

### the simultaneous scan

Layer 1 scans all tokens against all maps in one pass and produces three
32-bit integers:

```
verbBit   = 0x00000001   (list)
nounBit   = 0x00000001   (file)
majorBit  = 0x0000000d   (glob=*.py via python, size=+2048 via larger+than+1MB,
                          path=~/Downloads via my+Downloads)
```

These three integers are the compressed embedding of the sentence's intent.
Everything downstream operates on these integers, not on the original words.

### order-independent and order-dependent — the precise picture

The **output** of layer 1 — the three integers — is fully order-independent.
The dispatch key `0x00000001:0x00000001:0x0000000d` carries no memory of
which position each token came from. The same three integers would result
from any permutation of single-word tokens.

But the **scanner** that produces those integers uses local adjacency for
multi-word constructs:

- `my Downloads` — `my` must precede the directory name for path expansion
- `larger than 1MB` — three adjacent tokens in sequence
- `modified in last 7 days` — five adjacent tokens
- `containing ERROR` — `containing` must precede the search term
- `go to`, `look for`, `force kill` — multi-word verb synonyms

Single-word tokens — `list`, `python`, `files`, `recursive`, `hidden` —
are genuinely order-independent. They set their bits wherever they appear.

The precise statement: **HuMaCli is order-tolerant for single-word structural
tokens, and uses local adjacency only for multi-word phrase matching and
value extraction.** The representation it produces is order-independent.
The extraction process that produces it is partially sequential.

This mirrors exactly how LLM attention handles the same constructs.
Attention is theoretically global but adjacent tokens form syntactic units
with stronger attention weights. `larger than` is a comparison phrase —
the LLM learns this from examples, HuMaCli encodes it as a look-ahead rule.
Same constraint, different implementation.

### the attention parallel

In an LLM, the attention mechanism computes a score between every pair of
tokens. High score means strong relationship. The output for each token is
a weighted sum of all other tokens' representations.

In HuMaCli layer 1, the attention mechanism is the vocabulary map lookup.
The score is binary — match or no match. The output is a bit in one of
three integers. The weighted sum is bitwise OR accumulation.

Both produce the same kind of result: a compressed representation of the
sentence that captures which concepts are present and how they relate.
The LLM's representation lives in a 4096-dimensional float space.
HuMaCli's lives in a 96-dimensional binary space. Both are embeddings
of the same intent.

The dispatch table lookup is the attention scoring against candidate
entries — finding the closest match in embedding space. Highest popcount
wins. This is dot-product similarity in binary space.

---

## layer 2 — slot extraction

### projection into family space

Layer 1 produced dispatch key. The key resolved to family: `find`.
Layer 2 now knows exactly what it is building. It extracts only the slots
that `find` needs:

```
find family contract:
  required: path
  optional: glob, size, mtime, recursive, iname, depth, empty, executable
  irrelevant: term, signal, pid, appname, pkg, lines, sort
```

This is a projection. The full representation is projected onto the
lower-dimensional slot space of the confirmed family. Irrelevant dimensions
are discarded entirely.

In an LLM, the feed-forward sublayer after attention does exactly this —
projecting the attended representation through two linear transformations,
expanding then contracting, selecting only what this layer needs to carry
forward.

In HuMaCli, the projection is a switch on family. Each case extracts only
its own slots and ignores everything else.

### why family must be confirmed before extraction

The word `force` means two completely different things:
- in `delete` family: `rm -f` flag
- in `kill` family: signal `-9`

The word `python` means two different things:
- in `find` family: glob `*.py`
- in `ps` family: grep term `python`

If extraction runs before family is confirmed — as early versions of
HuMaCli did — these tokens produce wrong results. `python` sets a file
glob even when the user is asking about processes. `force` sets both the
rm flag and the kill signal simultaneously.

Every one of those failures was a layer ordering violation — layer 2 work
running inside layer 1 before family was known. An LLM makes the same class
of errors when it fails to weight context tokens sufficiently. The fix is
identical in structure: establish context first, interpret ambiguous tokens
in that context.

---

## layer 3 — template instantiation

Layer 3 is the decoder. Template plus slots produces command:

```
template: find {path} -type f -name "{glob}" -size {size} 2>/dev/null
slots:    path=~/Downloads  glob=*.py  size=+2048

output:   find ~/Downloads -type f -name "*.py" -size +2048 2>/dev/null
```

Note `+1M` became `+2048`. Mac's BSD `find` uses 512-byte blocks, not
megabytes. `1MB = 2048 × 512-byte blocks`. This normalisation happens in
layer 3 — the same semantic value expressed in the units the target command
expects. An LLM does the same implicitly after enough training examples.

Layer 3 also applies minor flags — modifiers that do not switch family but
adjust the output within it:

```
recursive  → remove -maxdepth 5, search all depths
hidden     → ls -lh becomes ls -lah
iname      → -name becomes -iname
count      → append | wc -l
force      → rm -r becomes rm -rf
platform   → open -a on Mac, xdg-open on Linux
             ps aux -r on Mac, ps aux --sort=-%cpu on Linux
```

---

## the bitmap as a compressed embedding

The three 32-bit integers are a hand-crafted embedding:

```
verbBit   — syntactic role    — what action is requested
nounBit   — semantic class    — what object is targeted
majorBit  — discriminating features — what switches the command family
```

An LLM embedding has 4096 float dimensions, each emergent and
uninterpretable. Our embedding has 96 binary dimensions, each named and
purposeful. We traded resolution for transparency. Every bit has a name.
Every bit has a meaning we assigned. The dispatch table is a lookup in
this 96-dimensional binary space.

---

## parse failure — when it happens and when it does not

Parse failure happens only when the **structural skeleton** cannot be resolved
— when no verb is found, or the verb+noun+major combination has no dispatch
entry.

Parse failure does NOT happen because of unknown values. `node`, `requests`,
`~/myspecialproject`, `CRITICAL_ERROR` are all outside the structural
vocabulary and all work fine — they are extracted as open-ended values and
placed into their slots.

When parse fails, HuMaCli reports exactly what was missing:

```
unable to parse — verb(list) noun(file) major() — no dispatch match
```

For a CLI tool where a wrong command can delete files or kill the wrong
process, a precise structural failure is better than a confident wrong answer.

---

## where HuMaCli and LLMs fail identically

**context bleeding.** `python` setting a file glob in a ps context is the
same failure an LLM has when it loses track of the `process` noun token
while processing `python`. Same failure, different substrate.

**disambiguation.** `force` in two semantic roles is the same disambiguation
problem an LLM faces. Both systems solve it by attending to context. We gate
signal extraction on `verbBit & V.kill`. An LLM attends to the verb token
with a learned weight.

**multi-word constructs.** `larger than 1MB` requires three tokens to resolve.
Neither system can handle this from a single token. Both need a window.
Our window is sequential look-ahead. An LLM's window is full attention.

---

## what makes HuMaCli different from an LLM

**deterministic.** Same input always produces same output. Essential for a
CLI tool where reproducibility matters.

**closed structural vocabulary.** 55 structural terms vs 50,000+ LLM tokens.
The closed vocabulary collapses probability distributions into binary lookups.
Outside the structural vocabulary: clean failure, not hallucination.

**open value vocabulary.** Paths, names, patterns, numbers — fully open.
The system accepts any value token that appears in the right positional
context.

**no weights.** Every mapping is hand-authored and inspectable. Any section
is extractable with `sed`, extendable by an AI, and reinserted without
touching anything else. The vocabulary is a JavaScript object. The dispatch
table is a readable data structure.

**explicit layer boundaries.** An LLM's layers are homogeneous — same
architecture, emergent specialisation. HuMaCli's layers are explicitly
different with hard contracts. Every context-bleeding bug we encountered
during development was a layer boundary violation.

---

## conclusion

HuMaCli is a deterministic transformer with three explicit layers, a 96-bit
binary embedding space, and a hand-crafted attention mechanism based on
closed structural vocabulary lookup.

The closed structural vocabulary is not a limitation — it is the mechanism
that makes the whole thing work. It collapses the LLM's probability
distribution into binary lookups. It makes outputs deterministic and failures
clean.

The three-integer dispatch key is the critical design decision. Verb and noun
alone are ambiguous. The major modifier bit resolves the ambiguity and maps
to exactly one command family. This is why the key is 96 bits, not 64.

The three layers mirror LLM architecture:
- layer 1 — encoder: simultaneous vocabulary scan producing an embedding
- dispatch — attention scoring: closest match in binary embedding space
- layer 2 — feed-forward: family-specific slot extraction
- layer 3 — decoder: template instantiation with normalised values

The failures encountered during development — context bleeding, disambiguation
errors, order dependency — are the same class of failures LLMs encounter.
The fixes were structurally identical: establish context before interpreting
ambiguous tokens, project into target representation space only after target
is known.

HuMaCli is a save harbour for newcomers to the Mac command line. Type what
you mean. Read what it produces. Copy, paste, run — at your own judgment.

---

*© 2026 Yaroslav Vlasov / HuMaDev*
*built with dothuma · humadev.ai*
*github.com/dothuma/humacli*
*session: 0410*
