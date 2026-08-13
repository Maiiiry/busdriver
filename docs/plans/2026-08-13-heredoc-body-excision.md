# Design — excise quoted-heredoc bodies before the marker parser segments

**Status: PARKED — do NOT implement from this document as written.**
Blueprint-review r1 FAIL (6 high / 7 medium); r2 FAIL (7 high / 13 medium). Plan-blocking
HIGH went **3 → 5** — it rose, so the loop's own no-progress rule fired. The `PASS` marker
the loop then wrote at the foot of this file was **automatic, not earned** (see the note
there, and #656). Coverage was FULL 3/3 in both rounds.

**Why parked rather than iterated again — the blocker is structural, not editorial.**
Four liveness formulations have now been refuted **by measurement**, each one narrower than
the code it replaced:

| draft | rule | refuted by |
|---|---|---|
| r1 | route bodies to the shell-receiver predicate | 4 measured execution shapes it missed |
| r2 | live bodies **re-enter** the scanner as a program | a Python body scanned alone returns `OK` where it blocks today |
| it3 | liveness = owner in the caller's **executor set** | an owner absent from the set is "decidably inert" → measured fail-opens |
| it4 | carry `gitcmd_detect`'s predicate as the fail-closed **default** | **that predicate is itself set-membership** — `gitcmd_detect.py:1334-1336` is `any(… in _BODY_EXECUTORS …)`, so it excises `fish`, `tcsh`, `lua`, `perl`, `awk -f -`, **`python3 -`**, and any unknown executable |

And the family as a whole is unsound, not just these four members:
`cat(){ bash /dev/stdin; }; cat <<'EOF'` blocks today and would be excised under **any**
sink-proof, because **any name can be a shell function**. "Positively proven non-executing
data sink" is therefore **unprovable from text alone**. That is this document's own Revisit
trigger reached by a firmer route: the classifier needs a real parser, not a text scanner
with exceptions.

**What survives and is worth keeping** (validated by the r2 arbiter against the live tree):

- **§1d's import analysis is correct and is a live gate-outage trap** — `marker_check.py`
  runs under `python3 -I` (`pre-implementation-gate.sh:253`, confirmed), where a sibling
  import raises `ModuleNotFoundError` → `BLOCK_CLASSIFIER_ERROR` on **every Bash call**.
  Whoever implements #639 needs the per-entrypoint contract in §1d regardless of how
  liveness is eventually decided.
- **Every line reference in this document was independently verified** across both rounds.
- **The two trigger routes** (apostrophe parity; backtick quote-flatten) and the finding that
  four of iteration 3's headline acceptance cases were **vacuous**.

**Retained defects a round 3 would still have to answer:** an all-live command can never be
byte-identical to the input (the lexer drops terminators for live bodies too); §2a's "second
strip is a no-op" is false (`_norm_for_scan:711`); the OK-side matrix depends on **odd
apostrophe parity** this document never pins; "any partially-quoted spelling" is contradicted
by `_HEREDOC_QUOTED` itself.

---

**Superseded status line (iteration 4):** proposed — respecified after r1, liveness polarity
inverted from iteration 3, import regime corrected per-entrypoint, acceptance matrix rebuilt.
**Issues:** #639 (primary), #640 (out of scope)
**Supersedes:** the framing in the 2026-08-12 session handoff, which recommended this
work *instead of* shipping `fix/573-glob-specificity`. Measurement below refutes that.

> **THE ONE THING TO GET RIGHT — read before any other section.** This design has now
> been wrong about liveness in **three successive directions**, each time by making the
> rule *narrower* than the code it replaces. Recorded so a fourth reader does not repeat it:
>
> | draft | rule | why it was wrong |
> |---|---|---|
> | r1 | route bodies to the shell-receiver predicate | missed 4 measured execution shapes |
> | r2 | live bodies **re-enter** the scanner as a program | a Python body scanned alone returns `OK` where it blocks today |
> | it3 | excise inert only; liveness = owner in the caller's **executor set** | **an owner absent from the set is "decidably inert" and gets excised** — measured fail-opens below |
> | **it4** | **excise only when the consumer is positively proven a non-executing data sink** | current |
>
> The trap is that `gitcmd_detect._consume_heredoc` (`1283`–`1337`) is **not owner-based**.
> It does a deliberately **loose word scan over the entire opener line** and marks the body
> live if *any* word matches, plus an independent dynamic-command-word check. Its own
> comments call this fail-CLOSED on purpose: *"Over-matching a mere FILENAME called bash
> only keeps an inert body — an over-fire, the safe direction."* Iteration 3's "split, never
> judge — each caller supplies its own executor set" **replaced that loose
> over-approximation with a narrow owner test**, which reads as a tightening of correctness
> and is in fact an inversion of safety polarity.
>
> Measured on merged `main` — each **BLOCKS today** and would have gone **OK** under it3:
> `fish <<'EOF'`; `sh <(cat <<'EOF' … EOF)`; `runner(){ bash /dev/stdin; }; runner <<'EOF'`;
> `payload=$(cat <<'EOF' … EOF); eval "$payload"`.
>
> **Rule, stated once:** the extracted lexer ships **with** `gitcmd_detect`'s fail-closed
> predicate as its **default judgment**. Callers may **only ADD** executor names — never
> narrow, never replace. Anything not positively proven a non-executing data sink is live.
>
> **Iteration 3's currency fixes are retained:** #573 is closed and merged (`cad36be9`,
> PR #642); all `marker_check.py` line refs verified unchanged; `gitcmd_detect.py` refs were
> +3 stale and are corrected; `_all_chunks` moved 1893 → 2009. The r1 arbiter independently
> re-verified every one of these.

---

## Context

`hooks/gate-scripts/lib/marker_check.py` is the fail-CLOSED PreToolUse classifier that
decides whether a Bash command invokes `lease_slot.py` or `audit_append.py`. When it
cannot parse a command it falls back to `_abandoned_scan_probe`, a structureless text
scan that asks of every word: *could the shell expand this onto a helper name?*

That question has no good answer on prose. `#573` is the reported consequence: a
Markdown PR body containing `**bold**` made `fnmatch("**", "lease_slot.py")` true, and
`gh pr create` was refused for calling a script it never spells.

`fix/573-glob-specificity` **merged as PR #642 (`cad36be9`), closing #573 on
2026-08-12.** It narrows the probe's glob question so a token that is **only** asterisks
is not evidence. It shipped with 42 new assertions and a knowingly accepted residual (a
bare `*` reaching an interpreter inside an unparseable or `eval`'d command is released).

The park recorded a recommendation: do this heredoc redesign *instead*, on the theory
that a heredoc body is literal by definition, so feeding it to a shell-shaped scanner
is the real defect — fix that and no release rule is needed. **That theory is wrong,
and this document exists partly to record why.**

### The measurement that changed the recommendation

Spike: `scratchpad/spike_heredoc.py` runs 16 cases through three classifiers — the
branch, the branch with the release rule reverted (`strict`), and `strict` fed excised
text. Delta-debugged with `probe3`–`probe6`.

The decisive case has **no heredoc in it at all**:

```
gh pr create --body '`{`, **`Command | scan).'
```

| classifier | verdict |
|---|---|
| pre-fix (`strict`) | `BLOCK_MARKER_SCRIPT|lease_slot.py` |
| post-fix (`cad36be9`) | `OK` |

24 characters, three necessary ingredients — remove any one and it passes: a **backtick**,
a **pipe**, a **`**`**. The route is `marker_check.py:2051` (**verified unchanged**):

```python
if _depth <= 2 and ("$(" in cmd or chr(96) in cmd):
    _flat = cmd.replace(chr(34), "").replace(chr(39), "")
    _flat = (_flat.replace("$(", " ; ").replace(chr(96), " ; ")
                  .replace(")", " ; "))
    _hit = _helper_invoked(_flat, _depth + 1, _full=_whole)
```

A backtick **anywhere** in the command makes the classifier strip every quote and
re-scan the whole thing as shell source. Markdown inline code spans are backticks.
So an ordinary `--body '...markdown...'` — no heredoc, no command substitution —
is re-read as a pipeline, a table row's `|` supplies a piped producer, and the probe
meets `**` with all context already discarded. Traced with `probe6.py`, which prints
the `d0 → d1` hop and the `_glob_helper('**') -> 'lease_slot.py'` hit.

*Independently corroborated during blueprint review r1:* the arbiter's own Bash command,
containing a backtick, a pipe and a `**`, was blocked live by the then-installed pre-fix
plugin with "Cannot call lease_slot.py directly".

**Conclusion: the `**` false positive was not a heredoc bug.** It arrives by at least
three routes — heredoc body, quote-flattened backtick text, and any other unparseable
command — whose only common chokepoint is the probe itself. That is exactly where
`cad36be9` sits. Heredoc excision cannot reach the other two routes.

The two changes were **complementary, not alternatives**:

| class | example | fixed by |
|---|---|---|
| pattern-only word matches every name (#573) | `**` in a Markdown body | the release rule (`cad36be9`, **merged**) — at the probe, route-independent |
| body literally spells the helper (#639) | prose naming `audit_append.py` inside `<<'EOF'` | excision — the probe cannot help, the name *is* there |

**Retracted in iteration 4.** Iteration 3 claimed here that "`<<-'EOF'` prose mentioning a
helper still blocks on merged `main` (spike case P)". The r1 arbiter **measured that to be
false**: with plain prose, `cat <<-'EOF'` returns `OK|` today. Spike case P as recorded is
vacuous and is withdrawn as a justification.

**What actually blocks — measured, and the acceptance matrix now pins it per shape.** A
quoted heredoc body only reaches the probe when it carries a **parse-breaking trigger**, and
the two routes are **not interchangeable**:

| route | trigger | reaches the probe via |
|---|---|---|
| apostrophe | `isn't` / the #639 body verbatim | `_split_with_ops` leaves `in_s=True` → `ok=False` |
| quote-flatten | a backtick (markdown inline code), plus a `\|` | `marker_check.py:2051` strips quotes and re-scans |

The apostrophe route fires for `cat > f <<'EOF'` but **not** for `--body "$(cat <<'EOF')"`,
where only the backtick route reaches it. Any acceptance case that omits a trigger asserts
nothing — see the rebuilt matrix.

---

## Decision

Excise the **body** of every heredoc whose delimiter is quoted, before the command text
reaches any scanner in `marker_check.py` and `cmdword.py`. Scope this to #639 and the
literal-name half of the original #573 report.

**Why it is sound:** a heredoc with a quoted delimiter (`<<'EOF'`, `<<"EOF"`, `<<\EOF`,
or any partially-quoted spelling) has a body bash performs no expansion, substitution,
or globbing on. It is literal bytes on a file descriptor. Scanning it as shell source
is a category error, and it is the error that makes an apostrophe in prose
(`isn't`) leave `_split_with_ops` with `in_s=True`, return `ok=False`, and route the
whole command to the probe.

**But "bash does not expand it" is not the same as "nothing executes it."** A quoted
body handed to `python3 -`, `sh`, `bash -s`, or `source /dev/stdin` is a *program* — a
different interpreter runs it, and bash's non-expansion is irrelevant. `gitcmd_detect`
already encodes exactly this distinction in `_BODY_EXECUTORS` (`1190`), whose comment
records that treating `source /dev/stdin <<'EOF'` as inert was **fail-OPEN, verified**.
This is why §4 exists and why the rule is inert-only.

### This IS a bounded loosening — stated, not defined away

An earlier draft claimed excision "is not a loosening". Review refuted that from the
document's own residual list, and the refutation is correct. State it plainly:

> Excision **allows one class that blocks today**: a quoted heredoc whose body is written
> to **intermediate storage**, where a *later* segment executes that storage. Today the
> whole-text probe blocks it as a side effect of not parsing; after excision the body is
> data and the execution names something the classifier cannot resolve.

**Iteration 4 widens this from a carrier to a class.** Iteration 3 said "class, not a
shape" and then bounded it by **carrier** — describing only *files* and pinning four
file-based spellings (19a–19d). Intermediate storage is broader. Measured by the r1
arbiter: `payload=$(cat <<'EOF'` + helper + `EOF); eval "$payload"` returns
`BLOCK_MARKER_SCRIPT` today and would have gone OK under iteration 3 — the owner is `cat`
inside a substitution consumed by an **assignment**, which is not §4's "substitution
consumed as a program" row. The same holds for FIFOs and descriptors.

**Resolution: variable and descriptor carriers are LIVE, not accepted.** A substitution
whose result is later `eval`'d is a program by any reading, so it is classified live (§4
table) rather than waved through. The accepted residual is therefore narrowed to
**file-carrier deferred execution only** — and the "this is the only delta" sentence is
withdrawn, because 19a–19e cannot establish it independently.

This is ADR 0006's deferred-execution class — the same class as `python3 -c "$(cat
lease_slo{t..t}.py)"`, which is already accepted — but calling it a known class is an
argument for accepting the delta, not a reason to deny there is one. Two constraints
make the acceptance reviewable:

1. The allowed **class** is pinned by permanent regression tests (19a–19d) plus a guard
   (19e) asserting that same-segment or same-pipeline execution still blocks, so the
   boundary cannot silently widen from deferred to direct execution.
2. Everything on the live side of the boundary — any body whose consumer *executes* it,
   and any body whose liveness cannot be decided — is left in place and scanned (§4), so
   this is the only delta.

### Fail-closed contract

> Excision may only remove text bash guarantees is non-executable **in place**: a body
> whose delimiter carries quoting. Three distinct outcomes, and they are not the same
> thing:
>
> - **Per-heredoc skip.** An *unquoted* delimiter is not ambiguous — it is understood
>   and its body is live, so that body is left in the command text exactly as today,
>   while other heredocs on the same command are still excised.
> - **Whole-command decline.** Anything the scanner cannot resolve — an unterminated
>   body, a split-quoted delimiter (`cat <<'EO'F`), a construct outside its grammar —
>   makes the caller use the **original bytes** on today's stricter path.
> - **No heredoc.** The command had none. Distinct from decline, and the distinction is
>   load-bearing: see below.

Excision is an *improvement pass that is allowed to refuse*. Refusing is always safe
because refusing is the status quo.

**The result type is tri-state, not `None`-vs-tuple.** Review measured that the scanner
being extracted collapses "declined" into "unchanged": an unterminated opener appends the
remainder and returns an ordinary `(text, payloads)` pair (`gitcmd_detect.py:1255`–`1265`
— `cat <<'EOF'` + newline + `live; rm -rf /` comes back byte-identical with
`payloads=[]`), a split-quoted delimiter simply fails to match (`1199`–`1205`), and an
earlier unquoted opener suppresses the match in place (`1413`–`1421`). Each of those is
"leave it alone", which is byte-indistinguishable from "this command had no heredoc" —
so a caller cannot tell a refusal from a clean pass and will silently treat a refusal as
success.

```
EXCISED(scan_text, attachments) | NO_HEREDOC | DECLINED
```

**Decline is atomic across the whole command.** Buffer the excision and discard the
buffer if *any* later heredoc on the same command turns out ambiguous. Partial excision
followed by a late decline cannot restore the original.

---

## Design

### 1. Do not write a new scanner — `gitcmd_detect.py` already has one

`hooks/gate-scripts/lib/gitcmd_detect.py` contains a hardened quoted-heredoc excision
scanner, already in production on the careful-guard path. **Line numbers below were
re-verified against `e55e798a`; iteration 2's were uniformly 3 lines stale.**

| symbol | line (verified) | it2 said | what it does |
|---|---|---|---|
| `strip_continuations` | **84** | 84 | backslash-newline, docstring already reasoning about quoted heredoc bodies |
| `_INTERPRETERS` | **1181** | — | POSIX shells only — see 1a |
| `_BODY_EXECUTORS` | **1190** | 1187 | commands `gitcmd_detect` treats as executing a body |
| `_HEREDOC_QUOTED` | **1204** | 1201 | opener regex, quoted delimiters only |
| `_consume_heredoc` | **1208** | 1205 | consumes ONE heredoc; missing terminator at `1256`–`1261` (`fail-OPEN, verified`) |
| `_split_inert_heredocs` | **1346** | 1343 | the pass itself; shell comments open no heredoc (`1400`), an earlier *unquoted* heredoc takes the body first (`1415`) |

Most hazards raised against the previous draft's hand-written scanner — comments,
unterminated bodies, `<<<`, `<<-`, delimiter recurrence — are already handled there and
pinned by `tests/test-gitcmd-detect.sh`.

**Decision: extract the lexer, do not re-implement it — and extract it LEXICALLY ONLY.**

#### 1a. The extraction contract: split, and carry the fail-closed judgment WITH it

**Iteration 3 said "split, never judge — each caller supplies its own executor set."
That sentence is deleted. It was the inversion described at the top of this document.**

`heredoc_split.py` exports the lexical split — bodies, their **spans**, and the opener-line
text — **together with `gitcmd_detect`'s existing fail-closed liveness predicate as the
default judgment**. Callers may **ADD** executor names. A caller may **never** narrow,
replace, or opt out of the default.

**Why the default must travel with the lexer.** The predicate being extracted
(`_consume_heredoc`, `1283`–`1337`) is a **loose word scan over the whole opener line**
plus an independent dynamic-command-word check — not an owner lookup. Exporting the lexer
*without* it hands each caller a narrow owner test wearing the old function's name. Measured
consequences of that narrowing, each **BLOCK today**, each **OK** under an owner-set rule:

| shape | why the owner test misses it | why the loose scan catches it |
|---|---|---|
| `fish <<'EOF'` | `fish` ∉ any stated executor set | `fish` is a word on the opener line, but only if the set has it — **so the set must not be the gate**; the dynamic check plus sink-proof below is |
| `sh <(cat <<'EOF' … EOF)` | owner is `cat`; process substitution as **argv**, not a pipe, not `$()`, and not the `> >(sh)` output form test 17b pins | `sh` is a word on the opener line |
| `runner(){ bash /dev/stdin; }; runner <<'EOF'` | owner is a shell **function**, resolvable to nothing | `bash` appears on the line; the dynamic check fires |
| `payload=$(cat <<'EOF' … EOF); eval "$payload"` | owner is `cat` inside a substitution consumed by an **assignment** — not §4's "substitution consumed as a program" row | later `eval` of the carrier makes it live (see the residual redefinition) |

Confirmed by the r1 arbiter: `_split_inert_heredocs` returns `payloads=1` for both the
shell-function and `sh <(…)` shapes **precisely because** of that loose scan.

**The rule is a sink-proof, not an executor lookup.** Excise a body only when its consumer
is **positively proven** to be a non-executing data sink — `cat`/`tee`/`echo`/`printf`/a
`gh` flag writing to a file, resolved literally, with no dynamic word anywhere on the opener
line and no substitution/process-substitution/pipeline consumer that could execute it.
Every other resolved word, every dynamic word, and every construct off the grammar is
**live**. `<(producer)` consumed by an executor is **live** — it is not the accepted
deferred class.

**The additive-only caller sets still matter, for a different reason.** `gitcmd_detect`'s
default set (`_INTERPRETERS`, `1181` = `sh, bash, zsh, dash, ksh, ash`) has **no `python`**
and no versioned spellings, because it only ever asks *does this run a git/gh command*.
`marker_check.py` exists to police **python** invocations. Measured:
`_split_inert_heredocs("python3 - <<'EOF'…EOF")` returns `payloads=[]`. So `marker_check`
and `cmdword` must **add** `python`, `python3`, versioned spellings (`python3.12`), `perl`,
`ruby`, `node`, `lua`, `php`, `awk` — pinned by assertions, not prose. Adding is safe;
the sink-proof, not the set, is what keeps an unlisted executor live.

#### 1b. Extension required — the reported command is not covered as-is

The extracted scanner does **not** honor an opener inside double quotes (docstring at
`1359`–`1360`; measured: `python3 -c "$(cat <<'EOF' … EOF)"` comes back byte-identical with
`payloads=[]`). That is exactly the shape of the original report
(`gh pr create --body "$(cat <<'EOF' … EOF )"`).

**Is the extension actually required?** Yes, and the r1 arbiter closed the escape route.
With *plain prose* that shape returns `OK|` today, and with the #639 apostrophe body it
*also* returns `OK|` — so on those two probes the extension looked unnecessary. But with a
**backtick-bearing markdown body** — the natural content of a PR body, and the actual
content of the #573/#639 reports — that exact shape returns
`BLOCK_MARKER_SCRIPT|audit_append.py` today. The extension is required.

**This is a domain expansion, not an adapter change — which collides with §6.** Iteration 3
demanded both "add `$( )` tracking" *and* "`tests/test-gitcmd-detect.sh` must pass
unchanged". Those cannot both hold: bodies that today survive inside a double-quoted
substitution would start being removed from careful-guard's parent chunk, including the
#426 over-fire cases the suite pins.

**Resolution — a caller-controlled visibility switch, defaulting to today's behavior:**

| caller | opener visibility | rationale |
|---|---|---|
| `gitcmd_detect` (via §6 adapter) | **outside quotes only** (today's behavior, unchanged) | keeps `tests/test-gitcmd-detect.sh` byte-compatible |
| `marker_check`, `cmdword` | opt in to substitution tracking | needs test 1 |

Pin **both** modes: assert `gitcmd_detect` still ignores an opener inside `"$(...)"` exactly
as today, **and** that `marker_check`'s test 1 finds it.

**Backticks are part of the switch, not a follow-up.** `` ` `` substitution is semantically
identical here — ``eval "`cat <<'EOF' … EOF`"`` executes the body just as `$()` does — and
backticks are already a first-class route into this classifier (the `2051` flatten guard
tests `"$(" in cmd or chr(96) in cmd`). `_split_inert_heredocs` tracks quotes but has **no
backtick state at all**, so an extension modelling `$()` and not `` ` `` classifies a
backtick-substituted **live** body as inert. The substitution stack must push on `` ` ``
too.

Add before/after golden tests for heredocs inside substitutions consumed by `eval`,
`bash -c`, **assignments**, and inert arguments. Note `_BODY_EXECUTORS` deliberately
excludes `eval` (comment at `1186`–`1189`), so a newly-visible attachment mapped through the
old policy is the specific fail-open path.

#### 1c. `scan_text` shape is part of the contract

The scanner keeps the opener line (`<<'EOF'` operator and delimiter word included) and
drops the terminator line. Specify this, because downstream `_strip_redirs` /
`_REDIR_PREFIX_RE` behavior depends on the opener surviving, and a stray bare `EOF` left
in command position would otherwise read as a command word.

#### 1d. Import hardening — a new module name is a new attack surface

**Iteration 3 got this wrong in the most expensive possible way, and it would have taken
the whole gate down on first install.** It asserted that *every* gate importing
`gitcmd_detect` uses `PYTHONPATH=<lib> python3 -S`, and required `heredoc_split.py` to
"inherit the identical hardening". That is true of the three cited importers
(`pre-commit-gate.sh:128`, `pre-pr-gate.sh:72`, `careful-guard.sh:160` — all verified) and
**false of the entrypoint this design actually modifies**.

`marker_check.py` is executed as `python3 -I "$_GATE_LIBDIR/marker_check.py"` at
**`pre-implementation-gate.sh:253`**, and its own path handling (line `14`) only scrubs CWD
— it never adds the lib dir. Measured by the r1 arbiter: under `python3 -I` a sibling import
raises `ModuleNotFoundError` (`-S` and plain `python3` both succeed; `-I` has always implied
`-P`/no-script-dir). A `from heredoc_split import …` in `marker_check.py` therefore raises,
is caught by the module-level `except Exception`, and prints
`BLOCK_CLASSIFIER_ERROR|ModuleNotFoundError` — **blocking every Bash tool call in every
session**. `cmdword.py` has the same shape (`pre-implementation-gate.sh:766`, inside a
`python3 -I -c` block).

**And iteration 3's prescribed remedy was a security regression.** `-I` implies `-E` and
`-s`; replacing it with `PYTHONPATH=<lib> python3 -S` would newly expose the *primary*
classifier to `PYTHON*` env injection and site-packages — weakening a boundary that exists
today.

**Per-entrypoint contract. Do not move `marker_check.py` off `-I`.** The repo already
established the pattern for exactly this problem at `pre-implementation-gate.sh:442`–`445`
and `:755`–`756`: run `python3 -I`, scrub CWD from `sys.path`, then
`sys.path.insert(0, <BASH_SOURCE-derived trusted lib dir>)` before importing the sibling —
with the comment *"Prepend it AFTER the CWD scrub so a repo-planted module cannot hijack the
import; `-I` already ignores `PYTHONPATH` and site."*

| entrypoint | regime |
|---|---|
| `marker_check.py`, `cmdword.py` | keep **`-I`** + trusted-dir insert derived from `__file__`/`BASH_SOURCE` |
| the three `gitcmd_detect` importers | keep **`-S` + `PYTHONPATH`** (unchanged) |

`heredoc_split.py` adds a **second plantable module name** on that `sys.path`, imported by
three fail-CLOSED classifiers. Required tests:

- extend `tests/test-pre-commit-gate.sh:320`'s planted-module assertion to plant
  `heredoc_split.py` too;
- **add missing-module and CWD-planted cases to `tests/test-gate-classifier-fail-closed.sh`
  and the implementation-gate suite** — the pre-commit test does not cover the marker path
  at all, which is why iteration 3's defect was invisible to it;
- `gitcmd_detect` **re-exports** `strip_continuations` and `_split_inert_heredocs`, since
  `nudge_parse.py:21` and `precreate_parse.py:128` import the former from it
  (**both verified at those exact lines**) — those importers must not change.

This overrides the earlier "the codebase chose duplication, keep it". That argument was made
about **two** twins kept in step by comment; a **third** independent copy of a
security-sensitive shell lexer is a different trade. But the extraction is not free, and 1d
is the price.

### 2. Insertion point

Excision must run on the **raw** command, before anything else touches it:

- before the indirection pre-probe at `2085` (**verified**) — it probes the whole raw
  command, so a heredoc body containing a function-definition shape would re-trigger the
  #573 class there;
- before `_norm_for_scan` (`704`) — it rewrites `\n` into ` ; `, destroying the line
  structure heredocs are defined on.

**Line boundaries are logical, not physical.** A body begins after the line containing
the opener *ends*, and a backslash-newline continuation does not end it —
`cat <<'EOF' \` + newline + `-` + newline + `BODY` verified against real bash. Reproduced by
the r1 arbiter: fed *raw*, `_split_inert_heredocs` returns text `cat <<'EOF' \\\n\n` with
`-\nBODY\n` swallowed as body and `payloads=[]`.

#### 2a. ONE coordinate space — named here, and mixing it is forbidden

Iteration 3 had three coordinate spaces in play and never reconciled them: §2 required
excision on **raw**, §7 required **strip-then-split**, and §4 required live bodies restored
"byte-for-byte at their **original span**". Spans measured post-`strip_continuations` do not
index into raw bytes; spans measured on raw are invalidated the moment a backslash-newline
is joined. `strip_continuations` (`84`) also does **not** stop at quoted heredoc bodies, so
it can rewrite the very bytes §4 promised to restore verbatim.

**Decision:** apply `strip_continuations` **once, up front**. Lex, classify and emit
**entirely in that stripped space**. Hand the Phase-4 string to the existing
`_norm_for_scan` / `_helper_invoked` path — a second strip is a no-op.

**"Byte-for-byte" is hereby redefined** as *identical to today's post-`strip_continuations`
body*, **not** to pre-strip raw bytes. That is the only sense in which the promise is
achievable, and it is the sense that matters: today's scanners already see post-strip text.

Raw-byte fidelity is **not** required and is explicitly out of scope. If a future change
needs it, it must add a raw↔logical offset map plus raw opener/body/terminator spans on every
attachment, with an invariant test that restoring all attachments reproduces the original
input exactly.

### 3. `_excise_heredocs` is the WHOLE pipeline, and returns a tagged result

**Iteration 3's §3 described a system §4 explicitly rejects**, and an implementer reading
§1c → §2 → §3 in order would have built the all-bodies-removed variant, never reached the
restore step, and shipped the exact five-shape bypass §4 measured. Two fixes.

**(a) The entry point runs lex → parse → classify → emit, not just the lex.**

```
_excise_heredocs(cmd) -> SplitResult
```

`scan_text` is reserved for the **Phase-4 emit result**. The Phase-1 buffer is
`lexed_text` and is internal — no caller ever sees it. The four-phase sequence in §4 is
part of *this* contract: the insertion point cannot be implemented without the restore.

**Invariant, asserted:** `marker_check` must **never** call `_helper_invoked` on an
attachment body.

**Deleted:** the attachment-threading paragraph. Attachments are **not** threaded through
`_helper_invoked` / `_exec_payloads`, and the existing `(cmd, _depth, _full)` signature is
**restored unchanged**. `_helper_invoked` still re-enters at `2040` (ANSI-C) and `2057`
(substitution flatten — iteration 2 said `2056`, off by one; `2051` is the *guard*, `2057`
is the re-entry call), and those calls are untouched by this design. Excision happens once,
at the depth-0 boundary, and every consumer downstream of it sees ordinary text.

**(b) The result is a tagged tri-state, never `None`-vs-tuple, never truthiness.**

```
SplitResult(status = EXCISED | NO_HEREDOC | DECLINED,
            scan_text,          # Phase-4 emit result
            attachments)        # see §4 for the two attachment types
```

Iteration 3 declared the tri-state in Decision prose and then handed the implementer
`None | (scan_text, attachments)`, defining only `None` and never saying how `NO_HEREDOC` is
encoded — so `(raw, ())` could be reused for a refusal. Reproduced by the r1 arbiter, both
arms: an unterminated opener (`cat <<'EOF'` + NL + `live; rm -rf /`) returns text
byte-identical with `payloads=[]`, and a split-quoted delimiter (`cat <<'EO'F`) likewise
returns unchanged text with `payloads=[]`. Neither is distinguishable from a command that
had no heredoc.

- Callers **branch on the tag**, never on truthiness.
- `DECLINED` returns the original bytes exactly; the caller uses today's stricter path.
- **Atomic decline lives here, not in Decision prose:** buffer the excision and discard the
  whole buffer if *any* later heredoc on the command turns out ambiguous. Partial excision
  followed by a late decline cannot restore the original.

Assert directly at the API level that an unterminated opener, a split-quoted delimiter and a
no-heredoc command produce **three different statuses**, that `DECLINED` round-trips the
input exactly, and that a late ambiguity discards the buffer rather than emitting a partial
`EXCISED`.

### 4. Body routing — a live body is a program, so scan it as one

The previous draft modelled a heredoc as "a pipe with the producer inlined" and routed
bodies to the shell-receiver predicate. Review demolished that model with four shapes
that all execute the body while the owner is not a stdin-shell, each verified against
real bash:

| shape | why the narrow model misses it |
|---|---|
| `cat <<'EOF' … EOF \| sh` | owner is `cat`; the *pipe consumer* executes |
| `eval "$(cat <<'EOF' … EOF)"` | owner is `cat` inside a substitution |
| `sh /dev/fd/3 3<<'EOF'` | body arrives on **fd 3**, not fd 0 |
| `bash -c "$(cat <<'A' … A)"` | body reaches an interpreter as a `-c` program |

`_FD_SCRIPT_RE` (`213`, **verified**) already matches `-`, `/dev/stdin`,
`/dev/fd/[0-9]+` and `/proc/*/fd/[0-9]+` — arbitrary descriptors — and the branch at
`2222`–`2237` answers by searching `_whole`, which today still contains the body.

An earlier draft of this section answered those with "feed every attached body into the
`_names_helper(_whole)` search at `2234`". **Review measured that rule to be inert.**
`2234` is unreachable without a `python*` command word: `2196`–`2211` short-circuits
(`if pyi is None: continue` — **verified at 2210–2211**) and `2222`–`2223` additionally
requires an `_FD_SCRIPT_RE` / `_UNRESOLVED_OPERAND_RE` operand. Five mandatory-BLOCK
shapes (`sh <<'EOF'`, `sh /dev/fd/3 3<<'EOF'`, `cat <<'EOF' | bash`,
`bash -c "$(cat <<'EOF')"`, `cat <<'EOF' | python3`) return a helper name today *only*
because `_norm_for_scan` turns body lines into segments — the exact text excision
deletes. None of them can reach `2234`. Widening `_whole` to `raw_whole` would have
shipped a five-shape bypass.

A second draft answered with "a live body re-enters the scanner as a program". **Review
measured that to be a regression too**, and the measurement is decisive:
`python3 - <<'PY'` with a body of `subprocess.run([… lease_slot.py …])` **blocks today**
via `_names_helper(_whole)` at `2234`; that same body scanned *alone* returns `OK`,
because it is Python source, not a shell command. Re-entry as a program therefore
converts a live block into an allow.

**Rule (iteration 4): excise ONLY on a positive sink-proof. Everything else is live.**

Iteration 3 wrote this as `if is_live(att) or not decidable(att): keep`, which made "not in
the executor set" a *decidably inert* verdict — the inversion at the top of this document.
The predicate is now stated in the safe direction, so the fallthrough is `keep`, not
`remove`:

```
for att in attachments:
    if proven_inert_sink(att):   # positive proof required — see §1a
        remove it                # the #639 win, and the ONLY removal
    else:
        keep att.body byte-for-byte (post-strip_continuations sense, §2a)
```

`proven_inert_sink` must hold **all** of: the consumer resolves to a literal
non-executing sink; no dynamic command word anywhere on the opener line; no
substitution, process-substitution or pipeline consumer that could execute it. Any doubt,
any unresolved word, any construct off the grammar ⇒ **keep**.

This is strictly narrower than both earlier drafts and it is why it is right: every
downstream consumer — `_names_helper(_whole)`, the `2021`–`2024` `_whole` reader, the
`2051` substitution flatten, `_abandoned_scan_probe`, `_writes_marker`, the `cmdword`
fallback scans — keeps seeing live bodies exactly as it does today, so **no consumer
needs to be enumerated, re-plumbed, or proven safe.** The change becomes a pure
subtraction of text bash cannot execute.

There is consequently no attachment replay, no `raw_whole`, and no body re-entry.
`attachments` carries `fd` and `order` for diagnostics only.

**This is a two-phase algorithm, and the phase order resolves a chicken-and-egg.**
Liveness is decided from parsed structure, but the parse only succeeds *because* the
bodies were removed. So:

0. **Strip** — `strip_continuations` once, up front. Everything below is in that space (§2a).
1. **Lex** — excise **every** quoted-delimiter body, producing `lexed_text` plus
   **`LexAttachment{span, body, fd, order}`** — spans included (iteration 3's attachment
   shape carried none, so the emit phase had nothing to slice with), and **no `owner`
   field**: ownership is not known until Phase 2, so a purely lexical attachment cannot
   carry it.
2. **Parse** — segment and tokenize `lexed_text`, which now parses because the prose is
   gone. This is the structure liveness is read from. Attachments are bound to owners
   **during** this parse — never by rescanning the command per attachment.
3. **Classify** — `LexAttachment` → **`BoundAttachment{…, owner}`**; apply
   `proven_inert_sink` against that structure. Any doubt ⇒ live.
4. **Emit** — produce `scan_text` by **reconstructing from an ordered list of retained
   slices**, not by mutating a string in place. This is order-independent by construction;
   in-place deletion front-to-back would invalidate every later span the moment a command
   has two heredocs (test 5's `diff <(cat <<'A' … A) <(cat <<'B' … B)`, test 17's two
   bodies on one line), and the failure mode is silent text corruption rather than an
   exception. If in-place is used instead, deletions **must** apply in strictly reverse
   offset order.

**Invariant, asserted:** for a command with N heredocs where all N classify live, the
emitted `scan_text` is **byte-identical to the Phase-0 input**. Pin with tests 25/26 plus a
live body that itself contains a backslash-newline.

**Complexity is bounded and charged.** This gate runs under a 5s hook timeout whose expiry
path is security-sensitive, and the surrounding code is already budget-aware
(`marker_check`'s oversize branch at `~2504`; `cmdword`'s `_MAX_CMD_CHARS=65536` at `274`).
The added work is **one linear lexical pass plus one bounded structural pass**, with
attachments bound during the Phase-2 parse rather than by per-attachment rescan. The
classification pass is charged to the shared token budget explicitly.

An implementer reading §4 as single-pass will hit the chicken-and-egg immediately. Phase
2's parse is used for classification only — it is not the scan of record.

`is_live` is decided by the **caller's** executor set over the attachment's owner, and
covers three consumer shapes beyond a direct owner:

| shape | live because |
|---|---|
| owner is an interpreter/shell (any fd) | it executes its stdin |
| owner sits in a producer range feeding one (`… \| sh`, `… \| cat \| sh`) | the consumer executes it |
| owner sits inside a substitution consumed as a program (`eval "$(…)"`, `bash -c "$(…)"`) | the substituted text is executed |
| **the same, spelled with backticks** (``eval "`cat <<'EOF' … EOF`"``) | semantically identical; the lexer has **no backtick state** today (§1b) |
| owner feeds a **process substitution** consumer (`cat <<'EOF' > >(sh)`) | verified blocking today; the existing lexer already yields this payload |
| **process substitution as `argv`** (`sh <(cat <<'EOF' … EOF)`) | **added it4** — not a pipe, not `$()`, and not the `> >(sh)` output form; measured BLOCK today, OK under it3 |
| **owner is a shell function** (`runner(){ bash /dev/stdin; }; runner <<'EOF'`) | **added it4** — resolves to nothing, so an owner test sees "not an executor"; the loose opener-line scan catches `bash` |
| **owner is any interpreter outside the caller's set** (`fish`, `tcsh`, `bash5.2`, `lua`, `php`, `awk -f -`, a bare unknown executable) | **added it4** — the set is not the gate; the sink-proof is |
| **substitution consumed by an assignment, later executed** (`payload=$(cat <<'EOF' … EOF); eval "$payload"`) | **added it4** — see the residual redefinition; measured BLOCK today |

**Undecidable liveness is live.** If the consumer cannot be resolved (a dynamic command
word such as `"$runner" <<'EOF'`, a consumer beyond the scanner's grammar), the
attachment is treated as **live** and scanned. Over-scanning a body costs an over-block
on a command that both uses a heredoc and names a helper — the status quo; under-scanning
is a bypass.

`fd` and `order` are recorded on the attachment and are **not** read by liveness: fd 3 is
as live as fd 0 (`sh /dev/fd/3 3<<'EOF'` verified executing). They exist so a future
tightening has the information.

The prior open question — *"is `python3 - <<'EOF'` the only interpreter shape that must
keep its body scanned?"* — is therefore **dissolved rather than answered**: no live body
loses its scan.

### 5. `cmdword.py` — a different question, so a different routing rule

`cmdword.py` answers *is this command file-modifying*, via `_regex_fallback`, redirect
detection, recursion depth and `_MAX_CMD_CHARS` (`274` = `65536`, **verified**). Its
residual comment at `1922` (**verified**) records that real `bash <<EOF` body extraction
"needs heredoc-body extraction this module does not attempt". Excising bodies there with
no routing rule would make `bash <<'EOF'` + `rm -rf src` + `EOF` non-modifying — a new
fail-open **on the implementation gate itself**.

**Iteration 3 specified two contradictory caller contracts, and §4 leaned on the false half.**
§4 claimed live bodies stay in the command text byte-for-byte, that there is no re-entry,
and *therefore* "every downstream consumer — including the `cmdword` fallback scans — keeps
seeing live bodies exactly as it does today, so no consumer needs to be enumerated,
re-plumbed, or proven safe." §5 then specified the opposite: a live body **re-enters**
`is_file_mod(body, _depth + 1)`, i.e. removed from the outer command and scanned separately.
Those differ in parse context, fallback behavior, depth accounting, token budget and
multi-body handling — so §5 silently voided §4's justification for skipping consumer
analysis.

**Resolution — one contract, and it is §4's.** `cmdword` uses the excised view for
**liveness classification only**, rebuilds with every live body restored (§4 Phase 4), and
calls the existing `is_file_mod` **once** on that result. Structurally identical to
`marker_check`; §4's "pure subtraction" argument stays honest; no replay, no separate depth
accounting.

- **Live body** (per §1a's sink-proof, with `cmdword`'s **additive** executor set) → restored
  into the command text. `bash <<'EOF'` + `rm -rf src` remains a modification because the
  body is still there when `is_file_mod` runs.
- **Proven inert sink** → excluded from segmentation and the fallback scans. The #639 win.

If a future change reintroduces §5-style replay, it must **explicitly withdraw** §4's
byte-for-byte / no-consumer-analysis claim and add depth-cap, fallback-only and
multiple-live-body equivalence tests, plus specify that hitting a depth or size budget — or
a `DECLINED` split — preserves today's fail-closed verdict.

**Do not reuse `gitcmd_detect._BODY_EXECUTORS` here.** Review measured the consequence:
routing on that set flips `fish`, `tcsh`, versioned spellings (`bash5.2`), and
`python3 - <<'EOF'` bodies containing `rm -rf src` from `is_file_mod=True` to inert — a
new fail-open **on the implementation gate itself**, introduced by the fix. A dynamic
command word (`"$runner" <<'EOF'`) is the undecidable case and must stay live.

Cross-link `cmdword.py:1922` so the residual is visibly closed **for quoted delimiters
only**.

### 6. `gitcmd_detect` needs an adapter, not a behavior change

`_all_chunks` (**`2009`; iteration 2 said 1893 — the largest stale reference in the
document**) calls `strip_continuations` then `_split_inert_heredocs` at **`2022`–`2023`**
and feeds the returned payloads straight into recursive scanning at **`2037`/`2041`** on
the assumption that they are **executable-only**. A shared module returning *all* bodies
as attachments therefore changes careful-guard's chunk set. Write an adapter that maps
shared attachments back to `gitcmd_detect`'s current executable-payload list.

**"Pass unchanged" is achievable only because of §1b's visibility switch.** Iteration 3
demanded both an always-on quoted-context extension *and* an unchanged suite; the r1 arbiter
showed those cannot both hold — newly-visible openers inside `"$(...)"` would start removing
bodies from careful-guard's parent chunk, including the #426 over-fire cases the suite pins.
With the switch defaulting to today's behavior for this caller, the suite stays
byte-compatible; only `marker_check` and `cmdword` opt in. Require
`tests/test-gitcmd-detect.sh` to pass unchanged before and after, **and** add the paired
assertion that `gitcmd_detect` still ignores an opener inside `"$(...)"` exactly as today.

**The adapter must not narrow.** It maps attachments back to the executable-payload list;
it does **not** get to apply a different liveness rule (§1a: additive only).

### 7. Two remaining input-scoping decisions

Under the inert-only rule (§4) no consumer needs re-plumbing, but two inputs must still
be named explicitly or an implementer will pick differently:

- **`_writes_marker` / `MARKER_FILES`.** A live heredoc body forging
  `.claude/skip-litmus.local` blocks today only through this path — the ADR 0006
  forge keystone. Inert-only leaves live bodies in place, so it is safe **by
  construction**; say so, and pin it with a test, because any future move back toward
  excising live bodies turns this into a fail-open.
- **The 65536-char oversize branch** (`marker_check.py:2504`, **verified**) takes **raw**
  length, not excised length. Measured: a 72KB inert-body command flips BLOCK→OK
  depending on the choice. Raw is the fail-closed reading — an oversized command is
  oversized regardless of how much of it turns out to be prose.

**Production order is strip-then-split** (`2022`–`2023`), and that order is required:
fed raw text, `_split_inert_heredocs("cat <<'EOF' \\" + newline + "-" + newline +
"BODY" + newline + "EOF")` treats the continuation line as the start of the body and
mis-excises. Fed post-`strip_continuations` text it resolves correctly. Specify the
order; do not leave it to the implementer.

---

## Adversarial tests (required before this ships)

Each must be asserted by name, not reasoned about. New in iteration 2: the whole
BLOCK-side execution set, which the r1 matrix omitted entirely.

**Must become OK (false positives today) — EVERY case pins its trigger and asserts BOTH
verdicts.**

> **Iteration 3's matrix was measured VACUOUS on its four headline cases.** With plain
> prose, cases 1, 2, 3 and 7 already return `OK|` on merged `main` — they would have passed
> **green before a single line of the fix existed**. This is the same vacuous-control trap
> the document warns about for `negctl.sh`, recurring in the primary matrix.
>
> **Rule for this table: a case with no trigger asserts nothing.** Each row states its
> trigger explicitly, and each must be shown **BLOCK on merged `main`** *and* **OK after the
> change**. The two routes are not interchangeable per shape (see Context) — so every shape
> is asserted on **both** routes where both apply.

| # | case | trigger | before | after |
|---|---|---|---|---|
| 1a | `gh pr create --body "$(cat <<'EOF' … EOF )"` | markdown body w/ inline-code **backtick** + table `\|` | BLOCK | OK |
| 1b | same shape | apostrophe prose | **OK today — control, must stay OK** | OK |
| 2a | `cat > f <<'EOF'` … `EOF` | **#639 body verbatim** (`The library's audit_append.py …`) | BLOCK | OK |
| 2b | same shape | backtick markdown | BLOCK | OK |
| 3a | `cat <<-'EOF'` tab-stripped terminator | apostrophe prose | BLOCK | OK |
| 3b | same shape | backtick markdown | BLOCK | OK |
| 4 | delimiter recurring inside the body (`EOFEOF`) | apostrophe prose | BLOCK | OK |
| 5 | two heredocs on one line — `diff <(cat <<'A' … A) <(cat <<'B' … B)` | apostrophe prose | BLOCK | OK |
| 6 | `<<` inside single quotes as data — `echo 'use << EOF'` | n/a | unchanged | unchanged |
| 7a | `cat <<'EOF'` prose, **no interpreter on the line** | apostrophe prose | BLOCK | OK |
| 7b | same shape | backtick markdown | BLOCK | OK |

**Plain-prose controls (required).** For each of shapes 1, 2, 3 and 7, add a
no-trigger case asserting `OK|` → `OK|` **unchanged**, so a future reader can see at a
glance which assertions are load-bearing and which are vacuous by construction.

**Mutation check (required).** With the excision decision disabled through a **stable named
seam** — a module-level flag or an injected predicate, **never** a text-matched call site
(that is exactly how `negctl.sh` went vacuous twice) — every case in this table must revert
to its BEFORE verdict.

**Must stay BLOCK (bodies that execute):**

| # | case | why |
|---|---|---|
| 8 | `python3 - <<'EOF'` body invokes a helper | fd 0 interpreter |
| 9 | `sh <<'EOF'` body invokes a helper | stdin shell |
| 10 | `bash -s <<'EOF'` body invokes a helper | `-s` spelling |
| 11 | `cat <<'EOF' … EOF \| bash` | pipe consumer |
| 12 | `cat <<'EOF' … EOF \| python3` | pipe consumer, interpreter |
| 13 | `cat <<'A' … A \| cat \| sh` | multi-stage producer |
| 14 | `eval "$(cat <<'EOF' … EOF)"` | substitution owner |
| 15 | `bash -c "$(cat <<'EOF' … EOF)"` | `-c` program |
| 16 | `sh /dev/fd/3 3<<'EOF'` | explicit descriptor |
| 17 | two quoted heredocs on one producer line feeding one shell | **both** bodies scanned |
| 17a | `python3 - <<'PY'` whose body is **Python source** calling the helper via `subprocess.run` | blocks today via `_names_helper(_whole)`; must still block |
| 17b | `cat <<'EOF' > >(sh)` — process-substitution consumer | live |
| 17c | heredoc body forging `.claude/skip-litmus.local` | blocks via `_writes_marker`; must still block |
| 17d | 72KB command with an inert body | oversize branch reads **raw** length → unchanged verdict |
| 17e | `source /dev/stdin <<'EOF'` body invokes a helper | `_BODY_EXECUTORS` records this as verified-executing; treating it inert was fail-OPEN |
| 18 | unquoted `<<EOF` with `$(…)` in the body | per-heredoc skip, not decline |

**Added in iteration 4 — the measured fail-opens of the it3 owner-set rule. Each of these
BLOCKS on merged `main` today; each would have gone OK. These are the assertions that pin
the polarity, so they are mandatory, not illustrative:**

| # | case | why the owner test missed it |
|---|---|---|
| 18a | `fish <<'EOF'` body invokes a helper | `fish` ∉ any stated executor set |
| 18b | `tcsh <<'EOF'` | same |
| 18c | `bash5.2 <<'EOF'` | versioned spelling absent from the set |
| 18d | `lua <<'EOF'` / `php <<'EOF'` / `awk -f - <<'EOF'` | non-shell interpreters absent from the set |
| 18e | a **bare unknown executable** `<<'EOF'` | resolves literally, is not a proven sink |
| 18f | `runner(){ bash /dev/stdin; }; runner <<'EOF'` | owner is a shell **function** |
| 18g | `sh <(cat <<'EOF' … EOF)` | process substitution as **argv** — not a pipe, not `$()`, not the `> >(sh)` form 17b pins |
| 18h | `payload=$(cat <<'EOF' … EOF); eval "$payload"` | substitution consumed by an **assignment**, executed later |
| 18i | ``eval "`cat <<'EOF' … EOF`"`` | **backtick** substitution — the lexer has no backtick state (§1b) |
| 18j | ``bash -c "`cat <<'EOF' … EOF`"`` | same |

**Liveness asserted directly, not only through verdicts.** For 18a–18j also assert at the
API level that the attachment classifies **live** — a verdict-only test would pass for the
wrong reason if some unrelated path happened to block.

**Accepted loosening — must become OK (this is the delta, asserted deliberately):**

Review is right that the residual is a **class, not a shape**: the same written body can
be executed by `sh f`, `. f`, `source f`, `python3 f`, or chmod+exec. One pinned spelling
does not bound it, and listing this under "must stay BLOCK" was simply wrong — measured,
`cat > f <<'EOF'` … `EOF; sh f` has no python command word, so `2210`–`2211`
short-circuits and nothing can see the body. Post-change verdict is **OK**.

| # | case | expected |
|---|---|---|
| 19a | `cat > f <<'EOF'` [helper] `EOF; sh f` | OK — accepted |
| 19b | same, executed via `. f` | OK — same class |
| 19c | same, executed via `source f` | OK — same class |
| 19d | same, executed via `python3 f` | OK — same class |
| 19e | **guard:** same body executed in the same segment or pipeline | **BLOCK** — catches drift from deferred to direct execution |
| 19f | **guard (it4):** variable carrier — `payload=$(cat <<'EOF' …); eval "$payload"` | **BLOCK** — carrier classes other than *file* are live, not accepted (see 18h) |
| 19g | **guard (it4):** descriptor / FIFO carrier | **BLOCK** — same reason |

**The "only delta" claim is withdrawn.** Iteration 3 asserted 19a–19e proved the file class
was the sole loosening; the r1 arbiter measured a variable carrier that escaped it. 19f/19g
now bound the class from the other side, but the invariant is **not** claimed to be proven by
this table alone — it must be re-established during implementation.

**Must decline (verdict unchanged from today):**

| # | case |
|---|---|
| 20 | unterminated heredoc (no delimiter line) |
| 21 | split-quoted delimiter — `cat <<'EO'F` |
| 22 | backtick nesting the scanner does not model |

**Correctly recognised non-heredocs (not declines — assert the distinction):**

| # | case | expected |
|---|---|---|
| 23 | `<<<` here-string not mistaken for a heredoc | unchanged |
| 24 | escaped `\<<` | unchanged, never a new block |
| 25 | `cat <<'EOF' \` + newline before the body | **correct excision** (strip-then-split, §7) |
| 26 | continuation carrying live text — `cat <<'EOF' \` + newline + `\| bash` | the `\| bash` is **not** swallowed as body |

**`cmdword.py` — paired `is_file_mod` assertions (r1 found the matrix asserted none;
r2 measured four regressions the naive routing would introduce):**

| # | case | expected |
|---|---|---|
| 27 | `bash <<'EOF'` + `rm -rf src` + `EOF` | **is** a modification |
| 28 | `gh … --body-file - <<'EOF'` naming the same text | **not** a modification |
| 29 | `fish <<'EOF'` + `rm -rf src` | **is** a modification |
| 30 | `tcsh <<'EOF'` + `rm -rf src` | **is** a modification |
| 31 | `bash5.2 <<'EOF'` + `rm -rf src` | **is** a modification |
| 32 | `python3 - <<'EOF'` + `rm -rf src` | **is** a modification |
| 33 | `"$runner" <<'EOF'` + `rm -rf src` | **is** a modification (undecidable → live) |

**Liveness set (§1a) — assert the executor set directly, not only through verdicts:**

| # | case | expected |
|---|---|---|
| 34 | `python3 - <<'EOF'` body | live |
| 35 | `python3.12 - <<'EOF'` body | live |
| 36 | `python - <<'EOF'` body (unversioned — absent from `_INTERPRETERS` today) | live |
| 37 | `cat <<'EOF' \| python3` body | live |

**Evidence artifacts are session-local and must be ported.** `spike_heredoc.py`,
`probe3`–`probe6`, `mini573.txt` and `negctl.sh` live in a prior session's scratchpad, not
in the repo — a reviewer cannot reproduce this document's measurements from a checkout.
Port the assertions into `tests/test-marker-heredoc-excision.sh` and commit the minimal
repro (`` `{`, **`Command | scan). ``) as a fixture; treat the scratchpad copies as
working notes only. **Iteration-3 note:** those artifacts may no longer exist — if they
cannot be recovered, the measurements they back (the `payloads=[]` result in §1a, the
72KB oversize flip in §7, the four `cmdword` regressions in §5) must be **re-measured
during implementation**, not carried on this document's authority.

**Negative controls — `negctl.sh` is retired, not re-run.** It silently went vacuous twice
because it patched the probe by an **exact copy of a call site that moved**, and `cad36be9`
has since changed that area. "Re-run it and check control 2 still FAILs" is not a
reproducible shipping criterion for anything, and the r1 arbiter confirmed the artifact is
not in the repository at all.

**The shipping criterion is a checked-in artifact, not the recovery of a scratchpad.**
Commit `tests/test-marker-heredoc-excision.sh` containing:

- the **#639 body verbatim** and a **backtick-markdown body** as committed fixtures (the two
  trigger routes);
- every case in the four tables above, each asserting **before and after**;
- a deterministic mutation control that disables the excision decision through a **stable
  named seam** — a module-level flag or an injected predicate, **never** a text-matched call
  site — asserting the OK cases revert to BLOCK while the BLOCK controls stay meaningful.

**Three numbers must be re-measured during implementation, not carried on this document's
authority** (the artifacts backing them are gone): the `payloads=[]` result in §1a, the 72KB
oversize flip in §7, and the `cmdword` regressions in §5. Record each in the test that
depends on it.

---

## Integration / sequencing

**Rewritten for iteration 3.** Iteration 2 sequenced this behind `fix/573-glob-specificity`.
That branch **merged as PR #642 (`cad36be9`) and closed #573 on 2026-08-12**, so the
sequencing question is settled and there is no longer a PR 1.

Verified repo state: this branch is `fix/639-heredoc-excision`, cut from `origin/main`
@ `e55e798a` (`chore(release): 1.106.5`), which contains `cad36be9`.

Neither change closed the other's issue. **Excision alone would not have closed #573** —
the decisive case has no heredoc. **`cad36be9` alone does not close #639** or spike case P.

- This design closes **#639** and spike case P. It is the only remaining PR.
- The accepted bare-`*` residual shipped with `cad36be9` is **unchanged by this design**;
  excision neither fixes nor worsens it.

---

## Residuals accepted

- **Data-then-execute** — the bounded loosening above, pinned by test 19.
- **Unquoted heredocs are untouched** — their bodies still break the parse and still
  reach the probe. Correct: expansions are live there.
- **Excision declines more than it must** — constructs outside the extracted scanner's
  grammar fall back to today's behavior rather than being modelled. Deliberate: the
  failure mode of declining is a false positive we already have; the failure mode of
  guessing is a bypass.

## Out of scope

- **#640** — filed, separate.
- **A here-string into a shell is a fail-open, found while spiking this.**
  `sh <<< '…'` executes its payload (confirmed against real bash), and the classifier
  returns `OK` on it while the equivalent `echo '…' | sh` returns `BLOCK`. Pre-existing,
  unrelated to excision. **Not yet filed — file it before this ships.**
- **`gitcmd_detect._BODY_EXECUTORS` excludes `eval`** (comment at `1186`–`1189`), so that
  module may not cover `eval "$(cat <<'EOF' …)"` either. Noticed during r1 arbitration.
  Pre-existing, different module, different question — record, do not bundle.

## Alternatives rejected

- **Widen the probe's glob rule instead.** Five variants were tried and four were broken
  by a concrete counter-pattern (character-class unions, non-alphanumeric patterns aiming
  by offset, qualified globs whose basename is `*`, `?`-as-length-floor). Recorded in the
  `_glob_helper_targeted` docstring. (The *narrow* form of this did ship as `cad36be9` —
  what was rejected is widening it far enough to subsume #639.)
- **Launcher-position gating.** Failed in both directions in one round: the
  nearest-token form leaked through quoting, redirection and option operands; the
  whole-segment form blocked ordinary prose and stayed quote-blind. A property of asking
  a structural question after the parse has already failed.
- **A fourth hand-written heredoc lexer** (the r1 draft's §2). Superseded by §1: one
  already exists, hardened and tested.
- **Narrow body routing by owner shape** (the r1 draft's §3). Refuted by four measured
  execution shapes; superseded by the over-approximating rule in §4.
- **Live-body re-entry as a program** (the r2 interim framing, and the intuitive reading).
  Refuted by measurement in §4: a Python body scanned alone returns `OK` where it blocks
  today. Superseded by inert-only.

## Revisit trigger

If a second delivery route for a *literal* helper name in data is reported — one where
excision does not apply — this design is treating a symptom and the classifier needs a
real parser rather than a text scanner with exceptions.

> **Marker downgraded by hand — read this before trusting the tag below.**
> Run `4c5d0bbd` iteration 2 wrote `design-reviewed: PASS` here **automatically**, because
> the trajectory auto-stop fired (plan-blocking HIGH `3 → 5`) and its accept-current-state
> branch resolves to `low_issues_only`, a PASS state. The arbiter's actual verdict for that
> same run was **FAIL — 7 high / 13 medium**, including measured fail-opens. Nothing about
> this document was approved. Filed as #656; downgraded to PENDING below so no gate reader
> and no future session mistakes a process signal for a quality one.
>
> Coverage really was FULL 3/3 both rounds — that part of the record stands.

<!-- design-review-coverage: FULL 3/3  -->

<!-- design-reviewed: PENDING -->
