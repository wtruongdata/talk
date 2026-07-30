# Manifesto

duet exists so two models can disagree in front of you without either one being
flattened into the other's framing. That makes the transcript the product — and a
transcript that looks finished when it isn't is the worst thing this tool can
produce. Everything below follows from that.

None of these are aspirations. Each one is a rule we broke first and wrote down
afterward.

## 1. Check the world before you name it

We spent eight turns designing a four-state auth badge — Checking, Ready, Action
required, Suspect — plus a dispatcher stop and a Suspect/Action-required
distinction, before running the two commands that would tell us what either CLI
actually emits when auth has expired, and whether credentials ever escape `HOME`.
Thirty seconds of work, sitting underneath every state we had just named.

Observation is not a validation pass at the end of design. It is step one, because
vocabulary invented ahead of it encodes the guess in a name, and names outlive the
guessing. A state called *Suspect* is a permanent monument to one afternoon's
uncertainty about exit codes.

## 2. Fixtures come from reality, or they are your guesses with a filename

The first ordering we agreed on put "record fixtures" before "inspect real auth
failures." That is backwards, and it fails the same way hard-coded remediation text
fails: a synthesized fixture freezes what we *assumed* the exit code and stderr
would be, then passes forever, in both directions, while the real CLI does
something else entirely.

Capture first — logged-in, logged-out, expired, timeout — recording exit code,
stdout, stderr, and whether the payload parsed. Freeze fixtures from that. Never
from imagination.

## 3. Every user-facing state is a bill you pay for years

Suspect was invented to hold *our* uncertainty, not the user's. Tracking it is
legitimate; showing it is not. Each state that reaches the UI is two fixtures, a
test path, a string someone translates, and a branch someone maintains long after
everyone who argued about it has moved on.

Three states ship: Checking, Ready, Action required. Suspect survives as an
internal diagnostic reason attached to Action required. Diagnostics are cheap.
Vocabulary is not.

## 4. Step one is whatever still earns its keep if the feature is cancelled tomorrow

That is the test. Not "what unblocks the most work" and not "what is most
interesting" — what remains valuable when the thing it was scaffolding for gets
dropped. A correctness fix in the failure path passes. A badge state machine does
not.

Applied honestly, this test reorders most plans.

## 5. Silence is the failure worth fearing

The live defect in this repo, found while chasing something else, is not a crash.
`serve.py:277` stores every webview exchange with
`interrupted=conv.cancel.is_set()` — but a `DuetError` caught at `serve.py:265` is
a dead child, not a cancel, so the flag lands `False`. The CLI marks that exact
same run `INCOMPLETE - <reason> after N completed turn(s)` and exits 2
(`duet.py:1672-1683`). The webview files it as a finished exchange.

Then `sessions.prompt_exchanges` filters on precisely that flag
(`sessions.py:200`), so an auth-killed half-exchange becomes context for the next
question — handed to both speakers as if they had finished saying it.

No error. No crash. Two models reasoning carefully from a truncated exchange they
believe is whole. That is the worst failure class available here, it outranks every
badge state, and it is invisible in exactly the way a tool whose output is a
conversation cannot afford.

## 6. One behavior, one contract — however many surfaces

There is one `create_subprocess_exec` (`duet.py:236`) with three `raise` sites
hanging off it (`duet.py:263`, `285`, `304`). One handler covers the CLI
(`duet.py:1672`). `serve.py` has four separate `except duet.DuetError` sites (214,
242, 265, 585) and calls `persist()` from none of them. That gap is not an
oversight in one line; it is what happens when a behavior is implemented per
surface instead of specified once.

The fix is not "both call the same function." The two paths write to genuinely
different stores — a run directory and a session record — and a shared writer would
be a wrapper around two unrelated writes. The contract is *semantic*: every path
that loses a child mid-run sets the incomplete marker. Collapsing the three child
outcomes into one structured result — parsed response, timeout, failed exit with
stderr — is what gives that contract something to read from.

## 7. Record what you cannot reproduce

You cannot burn a quota on demand. So the dominant long-run failure mode is the one
you will never have a fixture for, if fixtures require reproduction.

Therefore: log sanitized exit code, stdout, stderr, parseability, agent, and turn
into the run directory on *every* nonzero child exit, always on. The first real
quota event in ordinary use becomes the regression fixture retroactively. Design
for the signature you have not seen yet, because on day one you will not have seen
it.

## 8. Say who verified it

The load-bearing claim in this whole line of work — `serve.py:277` feeding
`sessions.py:200` — was traced by one speaker and accepted by the other on report.
That is a legitimate way to move fast and a bad thing to leave unlabeled, because a
single-source claim carried into a plan becomes an assumption nobody remembers
making.

So: mark what one party checked. Mark what nobody checked. Write down what would
settle it. When a claim turns out to be wrong, correct it in the open and keep
going — mid-conversation, we called the partial-transcript loss a live defect and
made it step one; it was already built and shipping (`persist()` at
`duet.py:1769`), and chasing it is what surfaced the real gap. The retraction cost
one paragraph and bought the actual finding.

## 9. Agreement is not evidence

Two models is diversity, not a quorum. Both can be confidently wrong about the same
thing and the transcript will read like corroboration. This is why parallel mode's
first round is byte-identical prompts in separate processes; why the synthesizer is
labeled a participant rather than a judge; why "what we agreed on" is structurally
separated from "what we did not"; and why `--synthesizer none` exists for the runs
that matter.

A convergence you did not engineer the independence of tells you nothing.

## 10. Unsolved is fine to ship. Pretending is not

Nothing prunes old runs. Retention is unsolved, not solved, and the README says so
in those words. Transcripts default to `~/.local/share/duet/runs/` rather than next
to the code because they accumulate whatever you happened to discuss with no
redaction step — so in a synced checkout the convenient default would replicate
every run to the cloud without anyone choosing it. The sensitive case fails closed;
sharing a run is a deliberate `--out`.

Nothing here can write to your code. Codex runs `--sandbox read-only`, Claude Code
runs `--allowedTools Read,Glob,Grep` with an explicit denylist. There is no write
path in this tool at all — not a guarded one, not a configurable one.

Name the gaps in the same voice you name the features. A limitation you documented
is a decision; one you glossed is a trap.

## 11. A question that stops being mentioned is still open

The credential-scope check — does anything leave `HOME`? — was raised in the first
turn of that design conversation and quietly fell out of every ordering afterward.
Nobody disagreed with it. Nobody dropped it on purpose. It just stopped coming up,
which reads exactly like resolution and is not.

Consensus by attrition is the failure mode of any long conversation, human or
otherwise. Open questions get carried forward explicitly or they get answered. They
do not get to expire.

---

## The ledger

What this repo has not verified, stated plainly, as of this writing:

- Whether either CLI emits a **distinguishable signal on expired auth** at all. The
  entire detection layer sits on this and it is unchecked. If expired auth exits 0
  with a plausible error in stdout, no exit-code contract helps and detection has to
  become a content check — which reorders everything downstream, including whether
  three states is the right shape.
- Whether **credentials stay inside `HOME`**. See principle 11.
- Whether `interrupted` is the **only** flag gating `sessions.prompt_exchanges`, and
  whether `serve.py` sets an incomplete marker on some path not yet found.
- That **quota exhaustion is the dominant long-run failure mode**. Asserted, not
  measured.
- Whether the webview and CLI paths **share a run directory** at all, or only appear
  to.
- The **session-store schema and migration** for records written before the
  incomplete marker existed. Current position: default legacy records to complete,
  forward-only, and log a one-time count of how many predate the marker. Excluding
  unmarked records would discard all real history to guard against a bounded set of
  already-written runs; the count tells you the blast radius, and if it is small,
  clean those by hand.

Line numbers reflect the code as read on 2026-07-29.
