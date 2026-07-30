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
#!/usr/bin/env python3
"""duet chat - an interactive chat box for talking to Fable and Sol together.

Modelled on the Claude Code / Codex TUI: a pinned input box at the bottom, output
scrolling above it, slash commands, input history, and Ctrl-C to interrupt a run
without killing the session.

Deliberately *not* a full-screen app. It uses an ANSI scroll region so your
terminal scrollback, selection and copy/paste keep working - the same choice
Claude Code makes.

Streaming reality, verified against claude 2.1.220 and codex-cli 0.146.0: Claude
emits token deltas, Codex sends its message in one piece. So in dialogue mode
Fable types out live and Sol lands complete. That asymmetry is shown honestly
rather than hidden behind a fake typewriter.

Standard library only.

    python3 chat.py --fake      # canned answers, zero calls
    python3 chat.py             # real
"""

from __future__ import annotations

import argparse
import asyncio
import json
import os
import queue
import re
import select
import shutil
import signal
import sys
import threading
import time
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path
from typing import Any, Callable

import duet

# ----------------------------------------------------------------------
# ansi
# ----------------------------------------------------------------------

ESC = "\x1b"
RESET = f"{ESC}[0m"
DIM = f"{ESC}[2m"
BOLD = f"{ESC}[1m"
FABLE_C = f"{ESC}[38;5;175m"
SOL_C = f"{ESC}[38;5;108m"
USER_C = f"{ESC}[38;5;110m"
WARN_C = f"{ESC}[38;5;179m"
ERR_C = f"{ESC}[38;5;167m"
RULE_C = f"{ESC}[38;5;240m"

BOX_HEIGHT = 4
MARK = "duet"  # placeholder for a logo
_SGR = re.compile(rf"{ESC}\[[0-9;]*m")
# One key or one escape sequence, for parsing a burst of input.
_KEYSEQ = re.compile(rf"{ESC}\[[0-9;]*[A-Za-z~]|{ESC}O[A-Za-z]")


def plain(text: str) -> str:
    return _SGR.sub("", text)


# ----------------------------------------------------------------------
# session state
# ----------------------------------------------------------------------


@dataclass
class Settings:
    mode: str = "parallel"
    profile: str = "software"
    personas: bool = True
    rounds: int = 1
    turns: int = 6
    first: str = "fable"
    synthesizer: str = "none"
    schema: bool = False
    repo: Path = field(default_factory=Path.cwd)
    context_patterns: list[str] = field(default_factory=list)
    claude_model: str = ""
    claude_effort: str = ""
    codex_model: str = ""
    codex_effort: str = ""
    timeout: int = 900

    def summary(self) -> str:
        bits = [self.mode, self.profile,
                f"{self.turns}t" if self.mode == "dialogue" else f"{self.rounds}r"]
        if not self.personas:
            bits.append("no-personas")
        bits.append(f"F:{self.claude_model or 'cfg'}/{self.claude_effort or 'cfg'}")
        bits.append(f"S:{self.codex_model or 'cfg'}/{self.codex_effort or 'cfg'}")
        if self.synthesizer != "none":
            bits.append(f"synth:{self.synthesizer}")
        if self.schema:
            bits.append("schema")
        return " · ".join(bits)


@dataclass
class Exchange:
    question: str
    replies: list[tuple[str, str, str]] = field(default_factory=list)
    synthesis: str = ""


# ----------------------------------------------------------------------
# screen: ANSI scroll region with a pinned input box
# ----------------------------------------------------------------------


def _raw_mode_modules() -> tuple[Any, Any]:
    """termios and tty, imported at use rather than at module scope.

    Only Screen.start/stop need them, and they do not exist on Windows. Importing
    them at the top meant serve.py - which imports this module for Settings,
    StreamResult and stream_speaker, none of which touch a terminal - could not be
    imported on Windows at all, so the VS Code backend died before printing
    anything. Deferring the import does not make the TUI work there; it stops the
    TUI's dependency from being everyone else's problem.
    """
    import termios
    import tty

    return termios, tty


class Screen:
    def __init__(self) -> None:
        self.rows, self.cols, self.region_bottom = 24, 80, 20
        self.out_col = 0
        self.status = ""
        self.buffer = ""
        self.cursor = 0
        self._saved: Any = None
        self._last_box = 0.0
        self._box_dirty = False
        self._pending = ""
        self.measure()

    def measure(self) -> None:
        size = shutil.get_terminal_size(fallback=(80, 24))
        self.cols = max(40, size.columns)
        self.rows = max(12, size.lines)
        self.region_bottom = self.rows - BOX_HEIGHT

    def start(self) -> None:
        self.measure()
        termios, tty = _raw_mode_modules()
        self._saved = termios.tcgetattr(sys.stdin.fileno())
        tty.setraw(sys.stdin.fileno())
        self.w(f"{ESC}[2J{ESC}[H")
        self.w(f"{ESC}[1;{self.region_bottom}r")
        self.w(f"{ESC}[{self.region_bottom};1H")
        self.out_col = 0

    def stop(self) -> None:
        self.w(f"{ESC}[r")
        self.w(f"{ESC}[{self.rows};1H\r\n")
        if self._saved is not None:
            termios, _ = _raw_mode_modules()
            termios.tcsetattr(sys.stdin.fileno(), termios.TCSADRAIN, self._saved)
            self._saved = None

    def w(self, text: str) -> None:
        sys.stdout.write(text)
        sys.stdout.flush()

    # ---- output ----

    def emit(self, text: str) -> None:
        """Append into the scrolling region, tracking the column ourselves.

        The terminal cursor cannot be queried cheaply, but the scroll region
        guarantees that after any write the cursor is on its last line - so only
        the column has to be tracked.
        """
        if not text:
            return
        self.w(f"{ESC}[{self.region_bottom};{self.out_col + 1}H")
        self.w(text.replace("\n", "\r\n"))
        for ch in plain(text):
            if ch in "\n\r":
                self.out_col = 0
            else:
                self.out_col += 1
                if self.out_col >= self.cols:
                    self.out_col = 0
        # Throttled: streaming arrives token by token, and redrawing the whole box
        # per token both flickers and floods the terminal.
        self.draw_box()

    def line(self, text: str = "") -> None:
        self.emit(text + "\n")

    def rule(self, label: str = "") -> None:
        width = max(4, self.cols - 2)
        if label:
            head = f"── {label} "
            self.line(f"{RULE_C}{head}{'─' * max(0, width - len(plain(head)))}{RESET}")
        else:
            self.line(f"{RULE_C}{'─' * width}{RESET}")

    def wrapped(self, text: str, prefix: str = "") -> None:
        width = max(20, self.cols - len(prefix) - 1)
        for para in text.split("\n"):
            if not para.strip():
                self.line(prefix.rstrip())
                continue
            current = ""
            for word in para.split(" "):
                if current and len(current) + 1 + len(word) > width:
                    self.line(prefix + current)
                    current = word
                else:
                    current = f"{current} {word}".strip()
            if current:
                self.line(prefix + current)

    # ---- input box ----

    def set_status(self, text: str) -> None:
        self.status = text
        self.draw_box()

    def draw_box(self, *, force: bool = False, interval: float = 0.06) -> None:
        """Redraw the pinned box, at most ~16x/sec unless forced.

        Without the throttle, one redraw per streamed token makes the box flicker
        and buries the actual output in escape sequences.
        """
        now = time.monotonic()
        if not force and now - self._last_box < interval:
            self._box_dirty = True
            return
        self._last_box = now
        self._box_dirty = False
        self._paint_box()

    def _paint_box(self) -> None:
        top = self.rows - BOX_HEIGHT + 1
        inner = self.cols - 2
        left = self.status[: max(0, inner - len(MARK) - 3)]
        pad = " " * max(1, inner - len(left) - len(MARK) - 2)
        avail = inner - 3
        shown = self.buffer[-avail:] if len(self.buffer) > avail else self.buffer
        cursor_col = 4 + min(self.cursor, len(shown))

        lines = [
            f"{DIM} {left}{pad}{MARK} {RESET}",
            f"{RULE_C}╭{'─' * inner}╮{RESET}",
            f"{RULE_C}│{RESET} {USER_C}>{RESET} {shown}{' ' * max(0, avail - len(shown))}{RULE_C}│{RESET}",
            f"{RULE_C}╰{'─' * inner}╯{RESET}",
        ]
        for index, content in enumerate(lines):
            self.w(f"{ESC}[{top + index};1H{ESC}[2K{content}")
        self.w(f"{ESC}[{top + 2};{cursor_col}H")

    def park(self) -> None:
        self.w(f"{ESC}[{self.region_bottom};{self.out_col + 1}H")

    # ---- keys ----

    def _fill(self, timeout: float | None) -> bool:
        """Pull whatever is available straight off the fd into our own buffer.

        Reading via buffered sys.stdin here would be a trap: read(1) slurps the
        rest of the burst into Python's userspace buffer, after which select() on
        the fd reports "nothing to read" and every remaining keystroke is lost.
        """
        if self._pending:
            return True
        fd = sys.stdin.fileno()
        ready, _, _ = select.select([fd], [], [], timeout)
        if not ready:
            return False
        try:
            data = os.read(fd, 4096)
        except OSError:
            return False
        if not data:
            self._pending += "\x04"  # EOF
            return True
        self._pending += data.decode("utf-8", "replace")
        return True

    def read_key(self, timeout: float | None = None) -> str | None:
        if not self._pending and not self._fill(timeout):
            return None
        first = self._pending[0]
        if first != ESC:
            self._pending = self._pending[1:]
            return first
        match = _KEYSEQ.match(self._pending)
        if match:
            self._pending = self._pending[match.end():]
            return match.group(0)
        # A lone ESC so far: give the rest of the sequence a moment to arrive.
        if len(self._pending) == 1 and self._fill(0.02) and len(self._pending) > 1:
            match = _KEYSEQ.match(self._pending)
            if match:
                self._pending = self._pending[match.end():]
                return match.group(0)
        self._pending = self._pending[1:]
        return ESC


# ----------------------------------------------------------------------
# streaming one speaker
# ----------------------------------------------------------------------


@dataclass
class StreamResult:
    text: str = ""
    cost_usd: float | None = None
    model: str = ""
    tokens: dict[str, Any] | None = None
    thread_id: str = ""
    error: str = ""
    # Codex emits several agent_message items per turn: narration first ("I'll
    # inspect the repository..."), then the real answer. Kept separate so they can
    # be rendered distinctly instead of glued into one run-on paragraph.
    items: list[str] = field(default_factory=list)


async def stream_speaker(
    speaker: duet.Speaker,
    prompt: str,
    *,
    repo: Path,
    timeout: int,
    fake: bool,
    on_delta: Callable[[str], None],
    cancel: threading.Event,
) -> StreamResult:
    result = StreamResult()

    if fake:
        body = duet._canned(speaker, prompt)
        for chunk in re.findall(r"\S+\s*", body):
            if cancel.is_set():
                result.error = "interrupted"
                return result
            on_delta(chunk)
            await asyncio.sleep(0.004)
        result.text, result.model = body, "fake"
        return result

    argv = speaker.stream_command(cwd=repo)
    env = {**os.environ, "NO_COLOR": "1", "TERM": "dumb", "PAGER": "cat"}
    try:
        proc = await asyncio.create_subprocess_exec(
            *argv,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=str(repo),
            env=env,
        )
    except OSError as exc:
        result.error = f"cannot start {speaker.kind}: {exc}"
        return result

    pieces: list[str] = []

    async def pump() -> None:
        assert proc.stdin is not None and proc.stdout is not None
        proc.stdin.write(prompt.encode())
        await proc.stdin.drain()
        proc.stdin.close()
        while True:
            raw = await proc.stdout.readline()
            if not raw:
                return
            line = raw.decode("utf-8", "replace").strip()
            if not line.startswith("{"):
                continue
            try:
                event = json.loads(line)
            except json.JSONDecodeError:
                continue
            _consume(event, speaker.kind, pieces, result, on_delta)

    task = asyncio.ensure_future(pump())
    deadline = time.monotonic() + timeout
    while not task.done():
        if cancel.is_set():
            task.cancel()
            _kill(proc)
            result.error = "interrupted"
            return result
        if time.monotonic() > deadline:
            task.cancel()
            _kill(proc)
            result.error = f"timed out after {timeout}s"
            return result
        await asyncio.sleep(0.04)

    try:
        await task
    except asyncio.CancelledError:
        pass
    await proc.wait()

    stderr = ""
    if proc.stderr is not None:
        stderr = (await proc.stderr.read()).decode("utf-8", "replace")
    if not result.text:
        result.text = "".join(pieces).strip()
    if not result.text and not result.error:
        result.error = (stderr or "no output").strip()[-400:]
    return result


def _kill(proc: Any) -> None:
    try:
        proc.kill()
    except (ProcessLookupError, OSError):
        pass


def _consume(
    event: dict[str, Any],
    kind: str,
    pieces: list[str],
    result: StreamResult,
    on_delta: Callable[[str], None],
) -> None:
    """Fold one JSONL event in. Shapes verified against both CLIs."""
    etype = event.get("type")

    if kind == "claude":
        if etype == "stream_event":
            inner = event.get("event") or {}
            if inner.get("type") == "content_block_delta":
                text = (inner.get("delta") or {}).get("text")
                if isinstance(text, str) and text:
                    pieces.append(text)
                    on_delta(text)
        elif etype == "result":
            final = event.get("result")
            if isinstance(final, str) and final.strip():
                result.text = final.strip()
            cost = event.get("total_cost_usd")
            if isinstance(cost, (int, float)):
                result.cost_usd = float(cost)
            usage = event.get("modelUsage")
            if isinstance(usage, dict) and usage:
                result.model = ", ".join(sorted(usage))
            if event.get("is_error"):
                result.error = str(event.get("subtype") or "claude reported an error")
        return

    # codex sends no token deltas; the message arrives whole
    if etype == "thread.started":
        result.thread_id = str(event.get("thread_id") or "")
    elif etype == "item.completed":
        item = event.get("item") or {}
        if item.get("type") == "agent_message":
            text = item.get("text")
            if isinstance(text, str) and text.strip():
                # Separate the items, or "...context." and "`rg` isn't installed"
                # end up as one sentence.
                separator = "\n\n" if pieces else ""
                pieces.append(separator + text.strip())
                result.items.append(text.strip())
                on_delta(separator + text.strip())
    elif etype == "turn.completed":
        usage = event.get("usage")
        if isinstance(usage, dict):
            result.tokens = usage
    elif etype in ("turn.failed", "error"):
        result.error = json.dumps(event)[:300]


# ----------------------------------------------------------------------
# the chat
# ----------------------------------------------------------------------

HELP = """Ask anything; Fable (Claude Code) and Sol (Codex) both answer.

  /mode parallel|dialogue   how they talk               (now: {mode})
  /profile software|general code question or anything   (now: {profile})
  /persona on|off           the two lenses              (now: {personas})
  /rounds N                 parallel: critique rounds   (now: {rounds})
  /turns N                  dialogue: total turns       (now: {turns})
  /first fable|sol          dialogue: who opens         (now: {first})
  /synth claude|codex|none  who writes a synthesis      (now: {synth})
  /schema on|off            structured JSON block       (now: {schema})
  /model NAME               Fable's model   ( /model sol NAME for Sol )
  /effort LEVEL             Fable's effort  ( /effort sol LEVEL for Sol )
  /repo PATH                directory both may read     (now: {repo})
  /context FILE             a file both receive verbatim
  /status  /save  /clear  /quit

  Ctrl-C interrupts a running answer   Up/Down input history   Ctrl-D quits"""


class Chat:
    pending_note = "working"

    def __init__(self, settings: Settings, *, fake: bool) -> None:
        self.s = settings
        self.fake = fake
        self.screen = Screen()
        self.history: list[str] = []
        self.hist_index = 0
        self.exchanges: list[Exchange] = []
        self.spend = 0.0
        self.calls = 0
        self.cancel = threading.Event()
        self.typed_ahead = ""
        self.eof_ahead = False
        self.fable = duet.Speaker(
            "Fable", "claude", "claude", settings.claude_model,
            duet.load_persona("fable", duet.PROFILES[settings.profile].fable_persona,
                              enabled=settings.personas, profile=settings.profile),
            settings.claude_effort,
        )
        self.sol = duet.Speaker(
            "Sol", "codex", duet.find_codex(None), settings.codex_model,
            duet.load_persona("sol", duet.PROFILES[settings.profile].sol_persona,
                              enabled=settings.personas, profile=settings.profile),
            settings.codex_effort,
        )

    # ---- helpers ----

    def refresh_speakers(self) -> None:
        self.fable.model, self.fable.effort = self.s.claude_model, self.s.claude_effort
        self.sol.model, self.sol.effort = self.s.codex_model, self.s.codex_effort
        prof = duet.PROFILES[self.s.profile]
        self.fable.persona = duet.load_persona(
            "fable", prof.fable_persona, enabled=self.s.personas, profile=prof.name)
        self.sol.persona = duet.load_persona(
            "sol", prof.sol_persona, enabled=self.s.personas, profile=prof.name)

    def idle_status(self) -> str:
        cost = f" · ${self.spend:.4f}" if self.spend else ""
        return f"{self.s.summary()} · {self.calls} calls{cost}"

    def context_block(self) -> str:
        try:
            return duet.load_context(self.s.context_patterns, None, self.s.repo).block(
                duet.PROFILES[self.s.profile])
        except duet.DuetError as exc:
            self.warn(str(exc).splitlines()[0])
            return ""

    def history_block(self) -> str:
        if not self.exchanges:
            return ""
        parts = [
            "## Earlier in this conversation",
            "",
            "The human asked these before, and you were both present for all of it.",
            "",
        ]
        for exchange in self.exchanges[-3:]:
            parts += [f"**Human:** {exchange.question}", ""]
            for _, who, text in exchange.replies:
                body = text if len(text) < 1200 else text[:1200] + " [...]"
                parts += [f"**{who}:** {body}", ""]
        return "\n".join(parts)

    def warn(self, text: str) -> None:
        self.screen.line(f"{WARN_C}! {text}{RESET}")

    def error(self, text: str) -> None:
        self.screen.line(f"{ERR_C}x {text}{RESET}")

    def note(self, text: str) -> None:
        self.screen.line(f"{DIM}{text}{RESET}")

    # ---- lifecycle ----

    def banner(self) -> None:
        mode = "FAKE MODE - canned answers, nothing invoked" if self.fake else \
               "your Claude and ChatGPT subscriptions - no API key"
        self.screen.line(f"{BOLD}[{MARK}]{RESET}{DIM}  Fable (Claude Code) + Sol (Codex){RESET}")
        self.screen.line(f"{DIM}  {mode}{RESET}")
        self.screen.line(f"{DIM}  /help for commands, /quit to leave{RESET}")
        self.screen.line()
        warning = duet.context_asymmetry(self.s.repo)
        if warning:
            self.warn(warning.splitlines()[0].strip())
            self.screen.line()

    def run(self) -> int:
        self.screen.start()
        signal.signal(signal.SIGINT, signal.SIG_IGN)
        try:
            self.banner()
            self.screen.set_status(self.idle_status())
            while True:
                queued = self.take_queued_line()
                if queued is not None:
                    text = queued
                    if text.strip():
                        self.screen.line(f"{USER_C}> {text}{RESET}{DIM}  (typed while busy){RESET}")
                elif self.eof_ahead:
                    break
                else:
                    text = self.read_line()
                    if text is None:
                        break
                text = text.strip()
                if not text:
                    continue
                self.history.append(text)
                self.hist_index = len(self.history)
                if text.startswith("/"):
                    if self.command(text) is False:
                        break
                else:
                    self.ask(text)
                self.screen.set_status(self.idle_status())
        finally:
            self.screen.stop()
        return 0

    def read_line(self) -> str | None:
        self.screen.buffer, self.screen.cursor = "", 0
        self.screen.draw_box(force=True)
        while True:
            key = self.screen.read_key(0.3)
            if key is None:
                before = (self.screen.rows, self.screen.cols)
                self.screen.measure()
                resized = (self.screen.rows, self.screen.cols) != before
                if resized:
                    self.screen.w(f"{ESC}[1;{self.screen.region_bottom}r")
                # Idle: only repaint if something actually changed, otherwise the
                # terminal is flooded with redraws while nothing happens.
                if resized or self.screen._box_dirty:
                    self.screen.draw_box(force=True)
                continue
            if key in ("\r", "\n"):
                value = self.screen.buffer
                self.screen.buffer, self.screen.cursor = "", 0
                self.screen.park()
                self.screen.line(f"{USER_C}> {value}{RESET}")
                return value
            if key == "\x04":
                return None
            if key == "\x03":
                self.screen.buffer, self.screen.cursor = "", 0
            elif key in ("\x7f", "\b"):
                if self.screen.cursor > 0:
                    buf = self.screen.buffer
                    self.screen.buffer = buf[: self.screen.cursor - 1] + buf[self.screen.cursor:]
                    self.screen.cursor -= 1
            elif key == "\x15":
                self.screen.buffer, self.screen.cursor = "", 0
            elif key == f"{ESC}[A":
                if self.history and self.hist_index > 0:
                    self.hist_index -= 1
                    self.screen.buffer = self.history[self.hist_index]
                    self.screen.cursor = len(self.screen.buffer)
            elif key == f"{ESC}[B":
                if self.hist_index < len(self.history) - 1:
                    self.hist_index += 1
                    self.screen.buffer = self.history[self.hist_index]
                else:
                    self.hist_index = len(self.history)
                    self.screen.buffer = ""
                self.screen.cursor = len(self.screen.buffer)
            elif key == f"{ESC}[C":
                self.screen.cursor = min(len(self.screen.buffer), self.screen.cursor + 1)
            elif key == f"{ESC}[D":
                self.screen.cursor = max(0, self.screen.cursor - 1)
            elif key.startswith(ESC):
                pass
            elif key.isprintable():
                buf = self.screen.buffer
                self.screen.buffer = buf[: self.screen.cursor] + key + buf[self.screen.cursor:]
                self.screen.cursor += len(key)
            self.screen.draw_box(force=True)

    # ---- slash commands ----

    def command(self, raw: str) -> bool | None:
        parts = raw.split()
        name, args = parts[0][1:].lower(), parts[1:]

        def need(count: int, usage: str) -> bool:
            if len(args) < count:
                self.error(f"usage: /{name} {usage}")
                return False
            return True

        if name in ("quit", "exit", "q"):
            return False
        if name in ("help", "h", "?"):
            for line in HELP.format(
                mode=self.s.mode, profile=self.s.profile,
                personas="on" if self.s.personas else "off",
                rounds=self.s.rounds, turns=self.s.turns,
                first=self.s.first, synth=self.s.synthesizer,
                schema="on" if self.s.schema else "off", repo=self.s.repo,
            ).split("\n"):
                self.note("  " + line)
            self.screen.line()
        elif name == "mode":
            if need(1, "parallel|dialogue"):
                if args[0] in ("parallel", "dialogue"):
                    self.s.mode = args[0]
                    self.note(f"  mode = {args[0]}")
                else:
                    self.error("mode must be parallel or dialogue")
        elif name in ("rounds", "turns"):
            if need(1, "N"):
                try:
                    value = int(args[0])
                except ValueError:
                    self.error("not a number")
                    return None
                if name == "rounds" and value >= 0:
                    self.s.rounds = value
                elif name == "turns" and value >= 1:
                    self.s.turns = value
                else:
                    self.error("out of range")
                    return None
                self.note(f"  {name} = {value}")
        elif name == "first":
            if need(1, "fable|sol") and args[0] in ("fable", "sol"):
                self.s.first = args[0]
                self.note(f"  first = {args[0]}")
        elif name in ("synth", "synthesizer"):
            if need(1, "claude|codex|none") and args[0] in ("claude", "codex", "none"):
                self.s.synthesizer = args[0]
                self.note(f"  synthesizer = {args[0]}")
        elif name == "profile":
            if need(1, "software|general"):
                if args[0] in duet.PROFILES:
                    self.s.profile = args[0]
                    self.refresh_speakers()
                    self.note(f"  profile = {args[0]}"
                              + ("  (no repo, no code vocabulary)" if args[0] == "general" else ""))
                else:
                    self.error(f"profile must be one of {', '.join(sorted(duet.PROFILES))}")
        elif name in ("persona", "personas"):
            if need(1, "on|off"):
                self.s.personas = args[0] in ("on", "true", "yes", "1")
                self.refresh_speakers()
                self.note(f"  personas = {'on' if self.s.personas else 'off'}"
                          + ("" if self.s.personas else "  (both get identical prompts)"))
        elif name == "schema":
            if need(1, "on|off"):
                self.s.schema = args[0] in ("on", "true", "yes", "1")
                self.note(f"  schema = {'on' if self.s.schema else 'off'}")
        elif name == "model":
            if need(1, "[sol] NAME"):
                if args[0] == "sol":
                    if need(2, "sol NAME"):
                        self.s.codex_model = args[1]
                        self.note(f"  Sol model = {args[1]}")
                else:
                    self.s.claude_model = args[0]
                    self.note(f"  Fable model = {args[0]}")
                self.refresh_speakers()
        elif name == "effort":
            levels = ("low", "medium", "high", "xhigh", "max")
            if need(1, "[sol] LEVEL"):
                if args[0] == "sol":
                    if need(2, "sol LEVEL"):
                        self.s.codex_effort = args[1]
                        self.note(f"  Sol effort = {args[1]}")
                elif args[0] in levels:
                    self.s.claude_effort = args[0]
                    self.note(f"  Fable effort = {args[0]}")
                else:
                    self.error(f"effort must be one of {', '.join(levels)}")
                self.refresh_speakers()
        elif name == "repo":
            if need(1, "PATH"):
                path = Path(args[0]).expanduser().resolve()
                if path.is_dir():
                    self.s.repo = path
                    self.note(f"  repo = {path}")
                else:
                    self.error(f"not a directory: {path}")
        elif name == "context":
            if need(1, "FILE"):
                self.s.context_patterns.append(args[0])
                block = self.context_block()
                if block:
                    self.note(f"  context: {len(self.s.context_patterns)} file(s), {len(block):,} chars")
                else:
                    self.s.context_patterns.pop()
        elif name == "status":
            self.note(f"  {self.s.summary()}")
            self.note(f"  repo:    {self.s.repo}")
            self.note(f"  context: {', '.join(self.s.context_patterns) or 'none'}")
            self.note(f"  so far:  {len(self.exchanges)} question(s), {self.calls} call(s)")
            if self.spend:
                self.note(f"  reported ${self.spend:.4f} (notional; subscriptions are not billed per call)")
        elif name == "save":
            self.save()
        elif name == "clear":
            self.exchanges.clear()
            self.note("  conversation forgotten (settings kept)")
        else:
            self.error(f"unknown command /{name} - try /help")
        return None

    # ---- asking ----

    def ask(self, question: str) -> None:
        self.refresh_speakers()
        if not self.fake:
            for speaker in (self.fable, self.sol):
                try:
                    speaker.resolve()
                except duet.DuetError as exc:
                    self.error(str(exc).splitlines()[0])
                    return

        self.cancel.clear()
        exchange = Exchange(question=question)
        started = time.monotonic()
        try:
            if self.s.mode == "dialogue":
                self.run_dialogue(question, exchange)
            else:
                self.run_parallel(question, exchange)
        except duet.DuetError as exc:
            self.error(str(exc))
        if exchange.replies:
            self.exchanges.append(exchange)
        if self.cancel.is_set():
            self.warn("interrupted")
        self.note(f"  {time.monotonic() - started:.0f}s · {self.calls} call(s) this session")
        self.screen.line()

    def poll_keys(self) -> None:
        """Read anything typed while a run is in flight.

        Ctrl-C interrupts. Everything else is queued and replayed afterwards, so
        typing ahead during a long answer is not lost.
        """
        while True:
            key = self.screen.read_key(0)
            if key is None:
                return
            if key == "\x03":
                if not self.cancel.is_set():
                    self.cancel.set()
                    self.screen.set_status("interrupting...")
                continue
            if key in ("\r", "\n"):
                self.typed_ahead += "\n"
            elif key in ("\x7f", "\b"):
                self.typed_ahead = self.typed_ahead[:-1]
            elif key == "\x04":
                self.eof_ahead = True
            elif not key.startswith(ESC) and key.isprintable():
                self.typed_ahead += key

    def take_queued_line(self) -> str | None:
        """Next complete line that was typed during a run, if any."""
        if "\n" not in self.typed_ahead:
            return None
        line, self.typed_ahead = self.typed_ahead.split("\n", 1)
        return line

    def pump(self, factory: Callable[[Any], Any]) -> Any:
        """Run an async job on a worker thread while the UI stays responsive."""
        box: dict[str, Any] = {}
        out: queue.Queue = queue.Queue()

        def worker() -> None:
            loop = asyncio.new_event_loop()
            try:
                box["value"] = loop.run_until_complete(factory(out))
            except BaseException as exc:
                box["error"] = exc
            finally:
                loop.close()
                out.put(("__done__", None))

        thread = threading.Thread(target=worker, daemon=True)
        thread.start()

        spin = "|/-\\"
        tick = 0
        last_spin = 0.0
        while True:
            # Poll input on every pass, not only when the queue drains. While a
            # fast stream is arriving the queue is never empty, and Ctrl-C would
            # otherwise go unnoticed until the run finished.
            self.poll_keys()
            try:
                kind, payload = out.get(timeout=0.05)
            except queue.Empty:
                now = time.monotonic()
                if now - last_spin > 0.12:
                    last_spin = now
                    tick += 1
                    self.screen.set_status(
                        f"{spin[tick % 4]} {self.pending_note}  ·  Ctrl-C interrupts"
                    )
                continue
            if kind == "__done__":
                break
            if kind == "emit":
                self.screen.emit(payload)
            elif kind == "status":
                self.pending_note = payload
        thread.join(timeout=2)
        self.screen.draw_box(force=True)
        if "error" in box:
            raise box["error"]
        return box.get("value")

    def run_parallel(self, question: str, exchange: Exchange) -> None:
        spec = duet.DEFAULT_SCHEMA if self.s.schema else None
        prof = duet.PROFILES[self.s.profile]
        context = "\n\n".join(x for x in (self.context_block(), self.history_block()) if x)

        def collect(prompts: dict[str, str], note: str) -> dict[str, StreamResult]:
            async def factory(out: Any) -> dict[str, StreamResult]:
                out.put(("status", note))
                done: dict[str, StreamResult] = {}

                async def one(speaker: duet.Speaker) -> None:
                    done[speaker.name] = await stream_speaker(
                        speaker, prompts[speaker.name], repo=self.s.repo,
                        timeout=self.s.timeout, fake=self.fake,
                        on_delta=lambda _t: None, cancel=self.cancel,
                    )
                    out.put(("status", f"{speaker.name} done, waiting for the other"))

                await asyncio.gather(one(self.fable), one(self.sol))
                return done

            return self.pump(factory) or {}

        results = collect(
            {
                "Fable": duet.opening_prompt(self.fable, question, repo=self.s.repo,
                                             other="Sol", spec=spec, context=context,
                                             profile=prof),
                "Sol": duet.opening_prompt(self.sol, question, repo=self.s.repo,
                                           other="Fable", spec=spec, context=context,
                                           profile=prof),
            },
            "both answering independently",
        )
        self.calls += 2
        for name in ("Fable", "Sol"):
            self.show(name, results.get(name), "Independent answer", exchange)
        if self.cancel.is_set():
            return
        latest = {name: (res.text if res else "") for name, res in results.items()}

        for turn in range(1, self.s.rounds + 1):
            prompts = {}
            for speaker, other in ((self.fable, "Sol"), (self.sol, "Fable")):
                prompts[speaker.name] = duet.exchange_prompt(
                    speaker, question, repo=self.s.repo, other=other,
                    other_text=latest.get(other, ""), own_text=latest.get(speaker.name, ""),
                    turn=turn, final_turn=turn == self.s.rounds, spec=spec, context=context,
                    profile=prof,
                )
            results = collect(prompts, f"exchange {turn} of {self.s.rounds}")
            self.calls += 2
            for name in ("Fable", "Sol"):
                self.show(name, results.get(name), f"Exchange {turn}", exchange)
            if self.cancel.is_set():
                return
            latest = {name: (res.text if res else "") for name, res in results.items()}

        self.maybe_synthesise(question, exchange)

    def run_dialogue(self, question: str, exchange: Exchange) -> None:
        context = "\n\n".join(x for x in (self.context_block(), self.history_block()) if x)
        order = [self.fable, self.sol] if self.s.first == "fable" else [self.sol, self.fable]
        history: list[tuple[str, str]] = []

        for turn in range(1, self.s.turns + 1):
            if self.cancel.is_set():
                return
            speaker, other = order[(turn - 1) % 2], order[turn % 2]
            colour = FABLE_C if speaker.kind == "claude" else SOL_C
            self.screen.rule(f"{speaker.name} · turn {turn}/{self.s.turns}")
            self.screen.emit("  " + colour)

            def factory(out: Any, speaker: duet.Speaker = speaker,
                        other: duet.Speaker = other, turn: int = turn) -> Any:
                async def job() -> StreamResult:
                    out.put(("status", f"{speaker.name} thinking, turn {turn}/{self.s.turns}"))
                    prompt = duet.dialogue_prompt(
                        speaker, question, repo=self.s.repo, other=other.name,
                        history=history, turn=turn, total=self.s.turns, context=context,
                        profile=duet.PROFILES[self.s.profile],
                    )
                    return await stream_speaker(
                        speaker, prompt, repo=self.s.repo, timeout=self.s.timeout,
                        fake=self.fake, on_delta=lambda t: out.put(("emit", t)),
                        cancel=self.cancel,
                    )
                return job()

            res = self.pump(factory)
            self.screen.emit(RESET)
            self.screen.line()
            self.calls += 1
            if res is None or res.error:
                self.error(f"{speaker.name}: {res.error if res else 'no result'}")
                return
            self.account(res)
            history.append((speaker.name, res.text))
            exchange.replies.append((f"Turn {turn}", speaker.name, res.text))

            tail = res.text.rstrip()[-40:]
            if duet.CONVERGED in tail or duet.IMPASSE in tail:
                marker = duet.CONVERGED if duet.CONVERGED in tail else duet.IMPASSE
                self.note(f"  {speaker.name} called {marker} - ending here")
                break

        self.maybe_synthesise(question, exchange)

    def maybe_synthesise(self, question: str, exchange: Exchange) -> None:
        if self.s.synthesizer == "none" or self.cancel.is_set() or not exchange.replies:
            return
        speaker = self.fable if self.s.synthesizer == "claude" else self.sol
        transcript = "\n\n".join(f"## {stage} - {who}\n\n{text}"
                                 for stage, who, text in exchange.replies)
        streams = speaker.kind == "claude"

        def factory(out: Any) -> Any:
            async def job() -> StreamResult:
                out.put(("status", f"{speaker.name} writing the synthesis"))
                return await stream_speaker(
                    speaker, duet.synthesis_prompt(question, transcript, synthesizer=speaker.name),
                    repo=self.s.repo, timeout=self.s.timeout, fake=self.fake,
                    on_delta=(lambda t: out.put(("emit", t))) if streams else (lambda _t: None),
                    cancel=self.cancel,
                )
            return job()

        self.screen.rule(f"Synthesis · {speaker.name}")
        colour = FABLE_C if speaker.kind == "claude" else SOL_C
        self.screen.emit("  " + colour)
        res = self.pump(factory)
        self.screen.emit(RESET)
        self.calls += 1
        if res is None or res.error:
            self.screen.line()
            self.error(f"{speaker.name}: {res.error if res else 'no result'}")
            return
        if not streams:
            self.screen.wrapped(res.text, prefix="  ")
        self.screen.line()
        exchange.synthesis = res.text
        self.account(res)

    def show(self, name: str, res: StreamResult | None, stage: str, exchange: Exchange) -> None:
        colour = FABLE_C if name == "Fable" else SOL_C
        self.screen.rule(f"{name} · {stage.lower()}")
        if res is None:
            self.error(f"{name}: nothing came back")
            return
        if res.error:
            self.error(f"{name}: {res.error}")
            return
        # Codex narrates before answering. Dim the narration so the answer stands
        # out, rather than running them together as one paragraph.
        if len(res.items) > 1:
            for note in res.items[:-1]:
                self.screen.emit(DIM)
                self.screen.wrapped(note, prefix="  · ")
                self.screen.emit(RESET)
            body = res.items[-1]
        else:
            body = res.text
        self.screen.emit(colour)
        self.screen.wrapped(body, prefix="  ")
        self.screen.emit(RESET)
        exchange.replies.append((stage, name, res.text))
        self.account(res)

    def account(self, res: StreamResult) -> None:
        if res.cost_usd:
            self.spend += res.cost_usd
        bits = [b for b in (res.model, f"${res.cost_usd:.4f}" if res.cost_usd else "") if b]
        if res.tokens and res.tokens.get("output_tokens"):
            bits.append(f"{res.tokens['output_tokens']} out tok")
        if bits:
            self.note("  " + " · ".join(bits))

    # ---- saving ----

    def save(self) -> None:
        if not self.exchanges:
            self.error("nothing to save yet")
            return
        stamp = datetime.now().strftime("%Y%m%d-%H%M%S")
        run_dir = duet.DEFAULT_OUT / f"{stamp}-chat-{duet.slugify(self.exchanges[0].question)}"
        run_dir.mkdir(parents=True, exist_ok=True)
        parts = [f"# duet chat - {stamp}", ""]
        for index, exchange in enumerate(self.exchanges, start=1):
            parts += [f"## {index}. {exchange.question}", ""]
            for stage, who, text in exchange.replies:
                parts += [f"### {stage} - {who}", "", text, ""]
            if exchange.synthesis:
                parts += ["### Synthesis", "", exchange.synthesis, ""]
        (run_dir / "chat.md").write_text("\n".join(parts), encoding="utf-8")
        self.note(f"  saved {run_dir / 'chat.md'}")


# ----------------------------------------------------------------------
# entry point
# ----------------------------------------------------------------------


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="chat.py",
        description="Interactive chat with Fable (Claude Code) and Sol (Codex) together.",
    )
    parser.add_argument("--mode", default="parallel", choices=["parallel", "dialogue"])
    parser.add_argument("--profile", default="software", choices=sorted(duet.PROFILES))
    parser.add_argument("--no-personas", action="store_true")
    parser.add_argument("--rounds", type=int, default=1)
    parser.add_argument("--turns", type=int, default=6)
    parser.add_argument("--first", default="fable", choices=["fable", "sol"])
    parser.add_argument("--synthesizer", default="none", choices=["claude", "codex", "none"])
    parser.add_argument("--schema", action="store_true")
    parser.add_argument("--repo", default=".")
    parser.add_argument("--context", action="append", metavar="FILE")
    parser.add_argument("--claude-model", default="")
    parser.add_argument("--claude-effort", default="",
                        choices=["", "low", "medium", "high", "xhigh", "max"])
    parser.add_argument("--codex-model", default="")
    parser.add_argument("--codex-effort", default="")
    parser.add_argument("--timeout", type=int, default=900)
    parser.add_argument("--fake", action="store_true", help="canned answers, zero calls")
    parser.add_argument("--selftest", action="store_true",
                        help="exercise the internals without a terminal")
    return parser


def selftest() -> int:
    """Check the parts that do not need a TTY."""
    ok = True

    screen = Screen()
    screen.cols, screen.rows = 60, 20
    screen.region_bottom = 16
    screen.status = "parallel · 0 calls"
    screen.buffer = "hello"
    screen.cursor = 5
    box = []
    screen.w = lambda text: box.append(text)  # type: ignore[method-assign]
    screen.draw_box()
    drawn = plain("".join(box))
    for expect in ("duet", "hello", "╭", "╰"):
        if expect not in drawn:
            print(f"FAIL box missing {expect!r}")
            ok = False

    payloads = [
        ("claude", {"type": "stream_event", "event": {"type": "content_block_delta",
                                                      "delta": {"type": "text_delta", "text": "Hi"}}}, "Hi"),
        ("claude", {"type": "result", "subtype": "success", "result": "final text",
                    "total_cost_usd": 0.5, "modelUsage": {"claude-opus-5": {}}}, ""),
        ("codex", {"type": "item.completed", "item": {"type": "agent_message", "text": "whole msg"}}, "whole msg"),
        ("codex", {"type": "turn.completed", "usage": {"output_tokens": 5}}, ""),
        ("codex", {"type": "thread.started", "thread_id": "abc-123"}, ""),
    ]
    for kind, event, expect_delta in payloads:
        pieces: list[str] = []
        got: list[str] = []
        res = StreamResult()
        _consume(event, kind, pieces, res, got.append)
        if expect_delta and expect_delta not in "".join(got):
            print(f"FAIL {kind} delta for {event['type']}")
            ok = False
    res = StreamResult()
    _consume(payloads[1][1], "claude", [], res, lambda _t: None)
    if res.text != "final text" or res.cost_usd != 0.5 or "opus" not in res.model:
        print("FAIL claude result event not folded in")
        ok = False
    res = StreamResult()
    _consume(payloads[3][1], "codex", [], res, lambda _t: None)
    if not res.tokens or res.tokens.get("output_tokens") != 5:
        print("FAIL codex usage not captured")
        ok = False
    res = StreamResult()
    _consume(payloads[4][1], "codex", [], res, lambda _t: None)
    if res.thread_id != "abc-123":
        print("FAIL codex thread id not captured")
        ok = False

    for kind in ("claude", "codex"):
        speaker = duet.Speaker("X", kind, "claude" if kind == "claude" else duet.find_codex(None))
        argv = speaker.stream_command(cwd=Path("/tmp"))
        needed = ["--output-format", "stream-json", "--include-partial-messages", "--verbose"] \
            if kind == "claude" else ["exec", "--json", "--sandbox", "read-only"]
        for flag in needed:
            if flag not in argv:
                print(f"FAIL {kind} stream_command missing {flag}")
                ok = False

    settings = Settings()
    chat = Chat(settings, fake=True)
    for command in ("/help", "/mode dialogue", "/turns 3", "/rounds 2", "/first sol",
                    "/synth claude", "/schema on", "/model opus", "/effort high",
                    "/model sol gpt-5.6-sol", "/effort sol high", "/status", "/clear",
                    "/nonsense", "/mode bogus", "/turns abc"):
        chat.screen.w = lambda text: None  # type: ignore[method-assign]
        try:
            chat.command(command)
        except Exception as exc:  # noqa: BLE001
            print(f"FAIL /{command} raised {exc!r}")
            ok = False
    if chat.s.mode != "dialogue" or chat.s.turns != 3 or chat.s.claude_effort != "high":
        print("FAIL slash commands did not apply")
        ok = False
    if chat.command("/quit") is not False:
        print("FAIL /quit should end the session")
        ok = False

    print("selftest: " + ("ok" if ok else "FAILURES"))
    return 0 if ok else 1


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)
    if args.selftest:
        return selftest()

    if not sys.stdin.isatty() or not sys.stdout.isatty():
        print("chat.py needs a terminal. For scripted use run duet.py, "
              "or try --selftest.", file=sys.stderr)
        return 2

    repo = Path(args.repo).expanduser().resolve()
    if not repo.is_dir():
        print(f"--repo is not a directory: {repo}", file=sys.stderr)
        return 2

    settings = Settings(
        mode=args.mode, profile=args.profile, personas=not args.no_personas,
        rounds=args.rounds, turns=args.turns, first=args.first,
        synthesizer=args.synthesizer, schema=args.schema, repo=repo,
        context_patterns=list(args.context or []),
        claude_model=args.claude_model, claude_effort=args.claude_effort,
        codex_model=args.codex_model, codex_effort=args.codex_effort,
        timeout=args.timeout,
    )
    try:
        return Chat(settings, fake=args.fake).run()
    except KeyboardInterrupt:
        return 130


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""duet - let Claude Code and Codex talk to each other about a question.

Two local CLIs, two different models, one neutral controller. Both run on your
existing subscriptions (Claude Pro/Max via Claude Code, ChatGPT via Codex), so
this spends plan allowance rather than API tokens.

    Question
      |- Fable (Claude Code) answers independently
      |- Sol   (Codex)       answers independently
      |- each critiques the other
      '- one of them writes the synthesis

The controller does not think. It hands out identical prompts, passes answers
between the two, caps the exchange, and writes a transcript. It never edits,
summarises, or takes a side.

Standard library only. Run `duet.py --fake "question"` to see the whole flow with
canned answers and zero calls.
"""

from __future__ import annotations

import argparse
import asyncio
import glob
import json
import os
import re
import shutil
import sys
import time
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path
from typing import Any

HERE = Path(__file__).resolve().parent
PERSONA_DIR = HERE / "personas"


def default_runs_dir() -> Path:
    """Where transcripts go by default: local, and deliberately not next to the code.

    Transcripts are the one artifact here that accumulates whatever you happened to
    discuss, with no redaction step and no retention policy. Writing them beside
    duet.py means that if the checkout lives in a synced folder - OneDrive, Dropbox,
    iCloud - every transcript replicates to the cloud and to every other synced
    device, without anyone choosing that. Defaulting somewhere local makes the
    sensitive case fail closed; `--out` is then the deliberate act of sharing one.

    Override with $DUET_RUNS. Pass --out to put a specific run anywhere you like.
    """
    override = os.environ.get("DUET_RUNS")
    if override:
        return Path(override).expanduser()
    base = os.environ.get("XDG_DATA_HOME")
    root = Path(base).expanduser() if base else Path.home() / ".local" / "share"
    return root / "duet" / "runs"


DEFAULT_OUT = default_runs_dir()

# Folders that replicate what you write to somebody else's servers.
SYNC_MARKERS = ("CloudStorage", "OneDrive", "Dropbox", "Google Drive", "iCloud Drive", "Box Sync")


def sync_warning(path: Path) -> str | None:
    """Warn when transcripts are about to land in a cloud-synced folder."""
    text = str(path)
    hit = next((m for m in SYNC_MARKERS if m in text), None)
    if hit is None:
        return None
    return (
        f"transcripts are being written inside a {hit} folder, so every run will "
        f"replicate to the cloud and to your other devices.\n"
        f"           There is no redaction or retention step. Use --out, or set "
        f"$DUET_RUNS, to keep them local."
    )

# Codex usually is not on PATH: it ships inside the ChatGPT app and the VS Code
# extension. Look there before giving up.
CODEX_CANDIDATES = (
    "/Applications/ChatGPT.app/Contents/Resources/codex",
    "~/.vscode/extensions/openai.chatgpt-*/bin/*/codex",
    "~/.vscode-insiders/extensions/openai.chatgpt-*/bin/*/codex",
)


class DuetError(Exception):
    """Something the user needs to fix, printed without a traceback."""


# ----------------------------------------------------------------------
# the two speakers
# ----------------------------------------------------------------------


@dataclass
class Speaker:
    """One side of the conversation, backed by a local CLI."""

    name: str  # persona name, e.g. Fable. NOT a model - see --claude-model.
    kind: str  # "claude" | "codex"
    bin: str
    model: str = ""  # "" means inherit from the CLI's own config
    persona: str = ""
    effort: str = ""  # "" means inherit
    add_dirs: tuple[Path, ...] = ()

    @property
    def label(self) -> str:
        return f"{self.name} ({self.kind})"

    def resolve(self) -> str:
        found = shutil.which(self.bin)
        if found:
            return found
        if Path(self.bin).is_file():
            return str(Path(self.bin).resolve())
        raise DuetError(
            f"{self.name}: cannot find `{self.bin}`.\n"
            + (
                "  Codex ships inside the ChatGPT app or the VS Code extension.\n"
                "  Point at it with --codex-bin, or symlink it:\n"
                "    ln -s /Applications/ChatGPT.app/Contents/Resources/codex ~/.local/bin/codex"
                if self.kind == "codex"
                else "  Install Claude Code, or point at it with --claude-bin."
            )
        )

    def command(self, prompt: str, *, cwd: Path, scratch: Path) -> tuple[list[str], str | None, Path | None]:
        """Return (argv, stdin, output_file). Prompt goes on stdin either way."""
        exe = self.resolve()
        if self.kind == "claude":
            argv = [
                exe, "-p",
                "--output-format", "json",
                "--allowedTools", "Read,Glob,Grep",
                "--disallowedTools", "Write,Edit,NotebookEdit,Bash,WebFetch",
            ]
            if self.model:
                argv += ["--model", self.model]
            if self.effort:
                argv += ["--effort", self.effort]
            for extra in self.add_dirs:
                argv += ["--add-dir", str(extra)]
            return argv, prompt, None

        out = scratch / f"sol-{int(time.time() * 1000)}.md"
        argv = [
            exe, "exec",
            "--sandbox", "read-only",
            "--skip-git-repo-check",
            "--color", "never",
            "--cd", str(cwd),
            "--output-last-message", str(out),
        ]
        if self.model:
            argv += ["--model", self.model]
        if self.effort:
            # Codex exposes reasoning effort through its config, not a flag.
            argv += ["-c", f'model_reasoning_effort="{self.effort}"']
        for extra in self.add_dirs:
            argv += ["--add-dir", str(extra)]
        argv.append("-")  # read the prompt from stdin
        return argv, prompt, out

    def stream_command(self, *, cwd: Path) -> list[str]:
        """argv for JSONL streaming, used by the chat UI.

        Verified against claude 2.1.220 and codex-cli 0.146.0:
        Claude emits token deltas (stream_event/content_block_delta); Codex does
        not - it sends the whole message at once in item.completed. The UI has to
        accommodate that asymmetry rather than pretend both stream.
        """
        exe = self.resolve()
        if self.kind == "claude":
            argv = [
                exe, "-p",
                "--output-format", "stream-json",
                "--include-partial-messages",
                "--verbose",  # required by the CLI for stream-json
                "--allowedTools", "Read,Glob,Grep",
                "--disallowedTools", "Write,Edit,NotebookEdit,Bash,WebFetch",
            ]
            if self.model:
                argv += ["--model", self.model]
            if self.effort:
                argv += ["--effort", self.effort]
        else:
            argv = [
                exe, "exec", "--json",
                "--sandbox", "read-only",
                "--skip-git-repo-check",
                "--color", "never",
                "--cd", str(cwd),
            ]
            if self.model:
                argv += ["--model", self.model]
            if self.effort:
                argv += ["-c", f'model_reasoning_effort="{self.effort}"']
        for extra in self.add_dirs:
            argv += ["--add-dir", str(extra)]
        if self.kind == "codex":
            argv.append("-")
        return argv


@dataclass
class Reply:
    speaker: str
    text: str
    seconds: float
    cost_usd: float | None = None
    model: str = ""
    payload: dict[str, Any] | None = None  # the structured block, when --schema is on
    repairs: int = 0


async def ask(speaker: Speaker, prompt: str, *, cwd: Path, scratch: Path, timeout: int, fake: bool) -> Reply:
    if fake:
        await asyncio.sleep(0.05)
        body = _canned(speaker, prompt)
        if "## Required structured block" in prompt:
            body += _canned_payload(speaker, prompt)
        return Reply(speaker.name, body, 0.05, None, "fake")

    argv, stdin, out_file = speaker.command(prompt, cwd=cwd, scratch=scratch)
    scratch.mkdir(parents=True, exist_ok=True)
    if out_file is not None:
        out_file.unlink(missing_ok=True)

    env = {**os.environ, "NO_COLOR": "1", "TERM": "dumb", "PAGER": "cat"}
    started = time.monotonic()
    proc = await asyncio.create_subprocess_exec(
        *argv,
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        cwd=str(cwd),
        env=env,
    )
    try:
        stdout_b, stderr_b = await asyncio.wait_for(
            proc.communicate(input=stdin.encode()), timeout=timeout
        )
    except asyncio.TimeoutError:
        proc.kill()
        await proc.wait()
        raise DuetError(f"{speaker.label} timed out after {timeout}s")
    seconds = time.monotonic() - started

    stdout = stdout_b.decode("utf-8", "replace")
    stderr = stderr_b.decode("utf-8", "replace")

    if speaker.kind == "claude":
        text, cost, model = _parse_claude(stdout, stderr, proc.returncode or 0)
    else:
        text, cost, model = _parse_codex(out_file, stdout, stderr, proc.returncode or 0)

    if not text.strip():
        raise DuetError(
            f"{speaker.label} returned nothing (exit {proc.returncode}).\n"
            f"  {(stderr or stdout).strip()[-600:]}"
        )
    return Reply(speaker.name, text.strip(), seconds, cost, model)


def _parse_claude(stdout: str, stderr: str, code: int) -> tuple[str, float | None, str]:
    body = stdout.strip()
    payload: Any = None
    if body:
        try:
            payload = json.loads(body)
        except json.JSONDecodeError:
            start = body.find("{")
            if start > 0:
                try:
                    payload = json.loads(body[start:])
                except json.JSONDecodeError:
                    payload = None
    if not isinstance(payload, dict):
        if code != 0:
            raise DuetError(f"claude failed (exit {code}): {(stderr or stdout).strip()[-600:]}")
        return body, None, ""  # structured output missing; keep the text anyway
    if payload.get("is_error"):
        raise DuetError(f"claude reported an error: {payload.get('subtype') or 'unknown'}")
    usage = payload.get("modelUsage") or {}
    model = ", ".join(sorted(usage)) if isinstance(usage, dict) and usage else str(payload.get("model") or "")
    cost = payload.get("total_cost_usd")
    return str(payload.get("result") or ""), (float(cost) if isinstance(cost, (int, float)) else None), model


_CODEX_NOISE = ("workdir:", "model:", "provider:", "sandbox:", "approval:", "reasoning", "tokens used", "codex", "user", "--------")


def _parse_codex(out_file: Path | None, stdout: str, stderr: str, code: int) -> tuple[str, float | None, str]:
    if out_file is not None and out_file.is_file():
        text = out_file.read_text(encoding="utf-8").strip()
        if text:
            return text, None, _sniff(stdout, r"model:\s*(\S+)")
    if code != 0:
        raise DuetError(f"codex failed (exit {code}): {(stderr or stdout).strip()[-600:]}")
    # Fall back to stdout with the run banner stripped.
    keep = [
        line for line in stdout.splitlines()
        if not any(line.strip().lower().startswith(p) for p in _CODEX_NOISE)
    ]
    return "\n".join(keep).strip(), None, _sniff(stdout, r"model:\s*(\S+)")


def _sniff(text: str, pattern: str) -> str:
    match = re.search(pattern, text)
    return match.group(1) if match else ""


# ----------------------------------------------------------------------
# prompts
# ----------------------------------------------------------------------

# ----------------------------------------------------------------------
# profiles
# ----------------------------------------------------------------------
#
# The software profile is what this started as: an architecture committee. Its
# prompts say "cite file.py:120", "argue from the code", and hand over a
# repository - all of which quietly narrow the scope. Ask it how to build a drone
# and both speakers dutifully grep your repo first.
#
# The general profile keeps the two lenses, which are domain-neutral (what breaks
# and who deals with it, versus what it takes to actually build the thing), and
# drops every software-specific instruction.


@dataclass(frozen=True)
class Profile:
    name: str
    evidence: str          # how to ground a claim
    argue_from: str        # what to reason from
    never_invent: str      # what fabrication looks like here
    context_authority: str # how context relates to what they can inspect
    show_repo: bool        # include the "repository you may read" section
    fable_persona: str
    sol_persona: str


SOFTWARE = Profile(
    name="software",
    evidence="Cite `path/to/file.py:120` or a named source.",
    argue_from="Argue from the question and, where relevant, from the code.",
    never_invent="Never invent file paths, numbers, benchmarks, or prior decisions.",
    context_authority="code, and say so explicitly if the code contradicts it.",
    show_repo=True,
    fable_persona="""\
Your instinct is to look at the system as a whole: boundaries, failure modes,
what happens under load or when a dependency dies, security and data handling,
and what this will cost to operate and maintain a year from now. You care about
what breaks and who gets paged.

Play to that instinct, but answer the actual question. Do not refuse to comment
on implementation detail if it matters.""",
    sol_persona="""\
Your instinct is to look at what it takes to actually build this: how it fits the
code that already exists, where the seams are, the concrete steps in order, what
the migration looks like, and what test or prototype would prove it works. You
care about whether it can be shipped.

Play to that instinct, but answer the actual question. Do not refuse to comment
on operational or design concerns if they matter.""",
)

GENERAL = Profile(
    name="general",
    evidence=(
        "Name the source: a measurement, a standard, a datasheet, a document, a "
        "regulation, a file and line - whatever the claim actually rests on."
    ),
    argue_from="Argue from the question and from what you actually know.",
    never_invent=(
        "Never invent numbers, sources, standards, part numbers, or prior "
        "decisions. If you are recalling something imperfectly, say so."
    ),
    context_authority=(
        "material below, and say so explicitly if what you can observe "
        "contradicts it."
    ),
    show_repo=False,
    fable_persona="""\
Your instinct is to look at the whole system and its life: where the boundaries
are, how it fails, what happens under stress or when one part gives out, safety
and regulatory exposure, and what it costs to run and maintain long after it is
finished. You care about what breaks and who has to deal with it.

Play to that instinct, but answer the actual question. Do not refuse to get
concrete about how the thing is made if that is what matters.""",
    sol_persona="""\
Your instinct is to look at what it takes to actually build the thing: the parts
and materials, what already exists that you can use, the concrete steps in order,
where the difficult joins are, and what test or prototype would prove it works.
You care about whether it can really be made, by these people, with what is at
hand.

Play to that instinct, but answer the actual question. Do not refuse to comment on
risk, operation or design if that is what matters.""",
)

PROFILES = {p.name: p for p in (SOFTWARE, GENERAL)}


def repo_block(profile: Profile, repo: Path) -> str:
    """The read-only repository section - only for profiles where code is the point."""
    if not profile.show_repo:
        return ""
    return f"\n## Repository you may read (read-only)\n\n`{repo}`\n"


def ground_rules(profile: Profile) -> str:
    return f"""\
## Ground rules

- You are one of two independent advisors, each a different model. Neither of you
  is in charge and neither reviews the other's work on their own authority. A
  human reads both of you and decides.
- Ground factual claims. {profile.evidence} If you have
  not verified something, say "I'm assuming" or "my estimate is" - never state it
  flatly as fact.
- {profile.never_invent}
- Do not agree to be agreeable, and do not manufacture disagreement to look
  rigorous. Say what you actually think.
- Be concrete and brief. Prefer specifics over hedged prose."""


# Kept for anything that imported it before profiles existed.
GROUND_RULES = ground_rules(SOFTWARE)


def opening_prompt(
    speaker: Speaker, question: str, *, repo: Path, other: str,
    spec: dict[str, str] | None = None, context: str = "",
    profile: Profile = SOFTWARE,
) -> str:
    tail = f"\n\n{schema_block(spec)}\n" if spec else ""
    ctx = f"\n{context}\n" if context else ""
    persona = f"\n{speaker.persona}\n" if speaker.persona else ""
    return f"""# Independent answer

You are **{speaker.name}**.
{persona}
{ground_rules(profile)}

## Your task

Answer the question below on your own terms.

The other advisor (**{other}**, a different model) is answering the same question
right now. You have not seen their answer and must not speculate about it. Do not
write "the other advisor will probably say...".
{profile.argue_from}

State plainly what you would need to know but do not, rather than filling the gap
with a guess presented as fact.
{repo_block(profile, repo)}{ctx}
## Question

{question}
{tail}"""


def exchange_prompt(
    speaker: Speaker,
    question: str,
    *,
    repo: Path,
    other: str,
    other_text: str,
    own_text: str,
    turn: int,
    final_turn: bool,
    spec: dict[str, str] | None = None,
    context: str = "",
    profile: Profile = SOFTWARE,
) -> str:
    tail = f"\n{schema_block(spec)}\n" if spec else ""
    ctx = f"\n{context}\n" if context else ""
    persona = f"\n{speaker.persona}\n" if speaker.persona else ""
    closing = (
        "This is the last exchange, so end with where you now stand and what is "
        "still unresolved between you."
        if final_turn
        else "There will be another exchange after this one."
    )
    return f"""# Exchange {turn}: respond to the other advisor

You are **{speaker.name}**.
{persona}
{ground_rules(profile)}

## Your task

**{other}** has answered the same question. Their response is below, after yours.

Work through it:

- Where are they right? Say so specifically. Conceding a good point is a strength.
- Where are they wrong, and why? Be precise about which claim you are rejecting.
- What are they assuming without evidence? Name it.
- Where do you now think you were wrong? Revise your own position and say what
  changed.
- For anything still contested: what observation, test, or measurement would
  settle it? If nothing could, say that - it means it is a judgement call, not a
  factual dispute.

Do not rewrite their answer for them, and do not restate your own answer whole.
Engage with what they actually said. {closing}
{repo_block(profile, repo)}{ctx}
## The question

{question}

---

## Your previous response

{own_text}

---

## {other}'s response

{other_text}
{tail}"""


# ----------------------------------------------------------------------
# project context
# ----------------------------------------------------------------------

CONVENTION_FILE = ".duet/context.md"
CONTEXT_WARN_CHARS = 80_000


@dataclass
class Context:
    """Text handed to both speakers verbatim, plus directories both may read.

    The point is symmetry. `--repo` only grants read access, and each CLI then
    picks up its *own* vendor file (Claude Code reads CLAUDE.md, Codex reads
    AGENTS.md), so a repo with only one of those briefs one speaker and not the
    other. Anything in here reaches both, byte for byte.
    """

    files: list[tuple[str, str]] = field(default_factory=list)  # (label, body)
    add_dirs: list[Path] = field(default_factory=list)
    warnings: list[str] = field(default_factory=list)

    @property
    def chars(self) -> int:
        return sum(len(body) for _, body in self.files)

    def block(self, profile: "Profile" = None) -> str:
        if not self.files:
            return ""
        authority = (profile or SOFTWARE).context_authority
        parts = [
            "## Project context",
            "",
            "The human provided the following. Both advisors received exactly this text,",
            "byte for byte. Treat it as authoritative over anything you infer from the",
            authority,
            "",
        ]
        for label, body in self.files:
            parts += [f"### {label}", "", body.strip(), ""]
        return "\n".join(parts)


def load_context(patterns: list[str] | None, add_dirs: list[str] | None, repo: Path) -> Context:
    """Resolve --context patterns (and the .duet/context.md convention) into text."""
    ctx = Context()
    seen: set[Path] = set()

    # (pattern, named_explicitly). A file you named by hand must not be skipped
    # quietly; a glob member or the convention file may be.
    candidates: list[tuple[str, bool]] = [
        (p, not any(ch in p for ch in "*?[")) for p in (patterns or [])
    ]
    convention = repo / CONVENTION_FILE
    if convention.is_file():
        candidates.insert(0, (str(convention), False))

    for pattern, named in candidates:
        expanded = os.path.expanduser(pattern)
        matches = sorted(glob.glob(expanded, recursive=True)) or ([expanded] if Path(expanded).exists() else [])
        if not matches:
            raise DuetError(
                f"--context matched nothing: {pattern}\n"
                "  Quote globs so duet expands them itself, e.g. --context 'docs/*.md'"
            )
        for match in matches:
            path = Path(match).resolve()
            if path in seen:
                continue
            if path.is_dir():
                raise DuetError(f"--context is a directory, not a file: {path}\n  Try --context '{path}/*.md'")
            try:
                body = path.read_text(encoding="utf-8")
            except UnicodeDecodeError:
                raise DuetError(f"--context file is not UTF-8 text: {path}")
            except OSError as exc:
                raise DuetError(f"--context cannot read {path}: {exc}")
            if not body.strip():
                # Never drop this quietly. A context file you named and that
                # contributes nothing is almost always an unsaved editor buffer.
                if named:
                    raise DuetError(
                        f"--context file is empty: {path}\n"
                        "  Nothing would reach either speaker. Unsaved editor buffer?\n"
                        "  Save the file, or drop the --context flag."
                    )
                ctx.warnings.append(f"skipped {path} - it is empty")
                continue
            seen.add(path)
            try:
                label = str(path.relative_to(repo))
            except ValueError:
                label = str(path)
            ctx.files.append((label, body))

    for raw in add_dirs or []:
        path = Path(os.path.expanduser(raw)).resolve()
        if not path.is_dir():
            raise DuetError(f"--add-dir is not a directory: {path}")
        if path not in ctx.add_dirs:
            ctx.add_dirs.append(path)
    return ctx


def context_asymmetry(repo: Path) -> str | None:
    """Warn when only one vendor's project file exists, briefing one speaker only."""
    claude_md = (repo / "CLAUDE.md").is_file()
    agents_md = (repo / "AGENTS.md").is_file()
    if claude_md == agents_md:
        return None
    have, missing, briefed = (
        ("CLAUDE.md", "AGENTS.md", "Fable") if claude_md else ("AGENTS.md", "CLAUDE.md", "Sol")
    )
    return (
        f"{have} exists but {missing} does not, so only {briefed} gets it automatically.\n"
        f"           Pass --context {have} to give both the same brief."
    )


# ----------------------------------------------------------------------
# structured output
# ----------------------------------------------------------------------

# Field -> type. Types: "str", "list[str]", "number", "confidence" (0.0-1.0).
DEFAULT_SCHEMA: dict[str, str] = {
    "recommendation": "str",
    "reasoning_summary": "list[str]",
    "assumptions": "list[str]",
    "risks": "list[str]",
    "confidence": "confidence",
    "evidence_needed": "list[str]",
}

_TYPE_HINTS = {
    "str": '"a single sentence"',
    "list[str]": '["one point", "another point"]',
    "number": "0",
    "confidence": "0.0",
}

_FENCE_RE = re.compile(
    r"^[ \t]*```[ \t]*(?P<lang>[A-Za-z0-9_+-]*)[ \t]*\n(?P<body>.*?)\n[ \t]*```[ \t]*$",
    re.DOTALL | re.MULTILINE,
)


def load_schema(path: str | None) -> dict[str, str]:
    """Schema spec from a JSON file, or the built-in default.

    Values may be type names ("str", "list[str]", "number", "confidence") or
    example values, in which case the type is inferred from them.
    """
    if not path:
        return dict(DEFAULT_SCHEMA)
    file = Path(path).expanduser()
    if not file.exists():
        raise DuetError(f"--schema-file not found: {file}")
    if not file.is_file():
        raise DuetError(f"--schema-file is not a regular file: {file}")
    try:
        raw = json.loads(file.read_text(encoding="utf-8"))
    except json.JSONDecodeError as exc:
        raise DuetError(f"--schema is not valid JSON: {exc}")
    if not isinstance(raw, dict) or not raw:
        raise DuetError("--schema must be a non-empty JSON object of field -> type or example")

    spec: dict[str, str] = {}
    for field_name, value in raw.items():
        if isinstance(value, str) and value in _TYPE_HINTS:
            spec[field_name] = value
        elif isinstance(value, list):
            spec[field_name] = "list[str]"
        elif isinstance(value, bool):
            raise DuetError(f"--schema: field {field_name!r} is a boolean; not supported")
        elif isinstance(value, (int, float)):
            spec[field_name] = "confidence" if 0 <= float(value) <= 1 else "number"
        else:
            spec[field_name] = "str"
    return spec


def schema_template(spec: dict[str, str]) -> str:
    body = ",\n".join(f'  "{name}": {_TYPE_HINTS[kind]}' for name, kind in spec.items())
    return "{\n" + body + "\n}"


def schema_block(spec: dict[str, str]) -> str:
    confidence_note = ""
    if any(kind == "confidence" for kind in spec.values()):
        names = [n for n, k in spec.items() if k == "confidence"]
        confidence_note = (
            f"\n- `{names[0]}` is a number from 0.0 to 1.0. Use it honestly: 0.9 means you "
            "would be surprised to be wrong, 0.5 means you are guessing. Do not default "
            "to 0.8 for everything."
        )
    return f"""## Required structured block

After your prose, as the **last thing in your response**, emit exactly one fenced
```json block with this shape and nothing else in it:

```json
{schema_template(spec)}
```

Rules:

- Every field is required. Use `[]` for a list with no entries rather than
  omitting it - silence should be explicit.
- Lists hold short strings, one idea each. Do not nest objects.
- The block must agree with your prose. If your prose hedges, the confidence must
  be low; if you name a risk above, it belongs in `risks`.
- `assumptions` is for things you did not verify. If you state something as fact
  in the prose without evidence, it belongs here instead.
- `evidence_needed` is what would actually settle the open questions - a
  measurement, a test, a document. Not "more research".{confidence_note}"""


def extract_payload(text: str) -> tuple[dict[str, Any] | None, str]:
    """Pull the last JSON object out of a response. Returns (payload, prose)."""
    found: list[tuple[re.Match[str], dict[str, Any]]] = []
    for match in _FENCE_RE.finditer(text):
        if (match.group("lang") or "").lower() not in ("", "json", "jsonc", "json5"):
            continue
        try:
            parsed = json.loads(match.group("body"))
        except json.JSONDecodeError:
            continue
        if isinstance(parsed, dict):
            found.append((match, parsed))
    if not found:
        return None, text.strip()
    match, payload = found[-1]
    prose = (text[: match.start()] + text[match.end() :]).strip()
    return payload, prose


def validate_payload(payload: Any, spec: dict[str, str]) -> list[str]:
    if not isinstance(payload, dict):
        return ["the block must be a JSON object"]
    errors: list[str] = []
    for name, kind in spec.items():
        if name not in payload or payload[name] is None:
            errors.append(f"`{name}` is missing (required)")
            continue
        value = payload[name]
        if kind == "str":
            if not isinstance(value, str) or not value.strip():
                errors.append(f"`{name}` must be a non-empty string")
        elif kind == "list[str]":
            if isinstance(value, str):
                errors.append(f"`{name}` must be a list of strings, not a single string")
            elif not isinstance(value, list):
                errors.append(f"`{name}` must be a list")
            else:
                for index, item in enumerate(value):
                    if not isinstance(item, str) or not item.strip():
                        errors.append(f"`{name}[{index}]` must be a non-empty string")
        elif kind in ("number", "confidence"):
            if isinstance(value, bool) or not isinstance(value, (int, float)):
                errors.append(f"`{name}` must be a number")
            elif kind == "confidence" and not 0.0 <= float(value) <= 1.0:
                errors.append(f"`{name}` must be between 0.0 and 1.0, got {value}")
    for extra in sorted(set(payload) - set(spec)):
        errors.append(f"`{extra}` is not in the schema; remove it")
    return errors


def repair_prompt(spec: dict[str, str], errors: list[str], previous: str) -> str:
    listed = "\n".join(f"- {e}" for e in errors)
    excerpt = previous[-3000:] if len(previous) > 3000 else previous
    return f"""# Fix your structured block

Your previous response was rejected: the JSON block at the end did not match the
required shape.

## What was wrong

{listed}

## What to do

Return the **same analysis and the same conclusions** with a corrected block. Do
not change your recommendation, lower your confidence, or drop a risk to make
validation pass - fix the structure only.

{schema_block(spec)}

## Your previous response

{excerpt}
"""


async def ask_structured(
    speaker: Speaker,
    prompt: str,
    *,
    spec: dict[str, str] | None,
    repairs: int,
    **kw: Any,
) -> Reply:
    """Ask, then enforce the schema, with a bounded number of repair attempts."""
    reply = await ask(speaker, prompt, **kw)
    if spec is None:
        return reply

    attempt = 0
    while True:
        payload, prose = extract_payload(reply.text)
        errors = (
            ["no ```json block found at the end of the response"]
            if payload is None
            else validate_payload(payload, spec)
        )
        if not errors:
            reply.payload = payload
            reply.text = prose or reply.text
            reply.repairs = attempt
            return reply
        if attempt >= repairs:
            # Keep the prose. A malformed block is worth flagging, not worth
            # discarding a real answer over.
            reply.payload = None
            reply.repairs = attempt
            reply.text = (
                reply.text
                + "\n\n> **Note from duet:** this response's structured block never "
                + f"validated ({len(errors)} error(s): {errors[0]}).\n"
            )
            return reply
        attempt += 1
        follow_up = await ask(speaker, repair_prompt(spec, errors, reply.text), **kw)
        follow_up.cost_usd = (reply.cost_usd or 0) + (follow_up.cost_usd or 0) or None
        follow_up.seconds += reply.seconds
        reply = follow_up


DIALOGUE_RULES = """\
## How this discussion works

- You are in a live, turn-taking discussion. The other speaker is a different
  model, running as a separate process. A controller relays each turn between you.
  Everything attributed to them below is genuinely theirs.
- **Write only your own turn.** Never write the other speaker's lines, never
  invent what they said or predict what they will say next. If you need something
  from them, ask for it and stop talking. Fabricating their side is the one thing
  that makes this whole exercise worthless.
- **One turn is a few sentences, not an essay.** Make one point, or answer the
  point just made. Leave room for a reply.
- Respond to what they actually just said. Name or quote the specific thing you
  are picking up.
- Do not restate your whole position each turn. The reader can see the history.
- Do not narrate the format ("as an AI", "in this turn I will"). Just talk."""

CONVERGED = "[CONVERGED]"
IMPASSE = "[IMPASSE]"

_STOP_RULE = f"""
- If there is genuinely nothing left to settle, end your turn with `{CONVERGED}`.
  If you have hit a real impasse where more talking will not help, end with
  `{IMPASSE}`. Use neither to close the conversation early out of politeness, and
  neither while a question you asked is still unanswered."""


def dialogue_prompt(
    speaker: Speaker,
    question: str,
    *,
    repo: Path,
    other: str,
    history: list[tuple[str, str]],
    turn: int,
    total: int,
    context: str = "",
    profile: Profile = SOFTWARE,
) -> str:
    """One turn of a sequential discussion, with the real history so far."""
    final = turn == total

    if not history:
        task = f"""## Your turn (turn 1 of {total})

You are opening. State your initial position in a few sentences - the shortest
version that {other} can actually push back on. Do not write an essay, and do not
try to cover everything; there are {total - 1} turns after this one."""
    else:
        last_speaker = history[-1][0]
        task = f"""## Your turn (turn {turn} of {total})

Respond to what **{last_speaker}** just said."""
        if final:
            task += f"""

This is the **final turn**. Close it out: say where you now stand, what {other}
changed your mind about if anything, and what is still unresolved between you."""

    lines = [
        f"# Discussion with {other} - turn {turn} of {total}",
        "",
        f"You are **{speaker.name}**.",
        "",
        *([speaker.persona, ""] if speaker.persona else []),
        DIALOGUE_RULES + ("" if final else _STOP_RULE),
        "",
        ground_rules(profile),
        "",
        *([f"## Repository you may read (read-only)\n\n`{repo}`", ""]
          if profile.show_repo else []),
        (context + "\n" if context else "") + f"## The question under discussion\n\n{question}",
        "",
    ]
    if history:
        lines.append("## The discussion so far\n")
        for index, (who, text) in enumerate(history, start=1):
            lines.append(f"**{who}** (turn {index}):\n\n{text}\n")
        lines.append("---\n")
    lines.append(task)
    return "\n".join(lines)


def synthesis_prompt(question: str, transcript: str, *, synthesizer: str) -> str:
    return f"""# Synthesis

You are **{synthesizer}**. You have just finished a conversation with another
advisor, a different model, about the question below. The full exchange follows.

Write the final answer for the human who asked.

You are one of the two participants, not an impartial judge, so hold to this:

- Do not quietly resolve a disagreement in your own favour. If the two of you did
  not settle something, say so and give both positions in their own terms.
- Separate **what you both agreed on** from **what remains contested**. A reader
  must be able to tell which is which at a glance.
- Keep every unverified assumption visible. Do not launder an estimate into a
  fact by restating it confidently.
- Where you disagreed, say what would settle it.
- Then give your recommendation, clearly labelled as yours, with your reasoning.

Structure it as:

1. **Short answer** - two or three sentences.
2. **What we agreed on**
3. **What we did not agree on** - both positions, and what would settle each.
4. **Assumptions this rests on**
5. **My recommendation** - and what would change my mind.

## Question

{question}

---

{transcript}
"""


# ----------------------------------------------------------------------
# the conversation
# ----------------------------------------------------------------------


@dataclass
class Transcript:
    question: str
    turns: list[tuple[str, Reply]] = field(default_factory=list)  # (stage, reply)
    synthesis: Reply | None = None
    note: str = ""  # e.g. [CONVERGED] or [IMPASSE], if a speaker called one
    spec: dict[str, str] | None = None

    def add(self, stage: str, reply: Reply) -> None:
        self.turns.append((stage, reply))

    def render(self, *, include_synthesis: bool = True) -> str:
        out = [f"# duet: {self.question}", ""]
        if self.note.startswith("INCOMPLETE"):
            # A run that died is not the same as a speaker choosing to stop, and
            # "a speaker called INCOMPLETE" would read as if one had.
            out += [f"> **{self.note}**", "",
                    "> The turns below are the ones that finished. Nothing after them ran.", ""]
        elif self.note:
            out += [f"_Ended early: a speaker called `{self.note}`._", ""]
        for stage, reply in self.turns:
            cost = f", ${reply.cost_usd:.4f}" if reply.cost_usd else ""
            out += [
                f"## {stage} - {reply.speaker}",
                "",
                f"_{reply.seconds:.0f}s{cost}"
                + (f", {reply.model}" if reply.model and reply.model != "fake" else "")
                + "_",
                "",
                reply.text,
                "",
            ]
            if reply.payload is not None:
                out += ["```json", json.dumps(reply.payload, indent=2), "```", ""]
        if include_synthesis and self.synthesis is not None:
            out += [f"## Synthesis - {self.synthesis.speaker}", "", self.synthesis.text, ""]
        return "\n".join(out)

    @property
    def total_cost(self) -> float:
        total = sum(r.cost_usd or 0 for _, r in self.turns)
        if self.synthesis is not None:
            total += self.synthesis.cost_usd or 0
        return total


async def discuss(
    a: Speaker,
    b: Speaker,
    question: str,
    *,
    turns: int,
    synthesizer: str,
    repo: Path,
    scratch: Path,
    timeout: int,
    fake: bool,
    log,
    context: str = "",
    profile: Profile = SOFTWARE,
    transcript: Transcript | None = None,
) -> Transcript:
    """Sequential turn-taking. One speaks, the other actually hears it, then replies.

    This is the only honest way to get a discussion out of two separate processes.
    Asked to converse in parallel, a model will invent the other's lines - so each
    turn here waits for the previous one and carries the real history.
    """
    # main owns this when it passes one in, so an interrupted run keeps its turns.
    transcript = transcript if transcript is not None else Transcript(question=question)
    history: list[tuple[str, str]] = []
    order = [a, b]
    stopped = ""

    for turn in range(1, turns + 1):
        speaker = order[(turn - 1) % 2]
        other = order[turn % 2]
        log(f"\n>> Turn {turn} of {turns}: {speaker.name}")
        reply = await ask(
            speaker,
            dialogue_prompt(
                speaker, question, repo=repo, other=other.name,
                history=history, turn=turn, total=turns, context=context, profile=profile,
            ),
            cwd=repo, scratch=scratch, timeout=timeout, fake=fake,
        )
        transcript.add(f"Turn {turn}", reply)
        history.append((speaker.name, reply.text))
        log(f"   {_preview(reply.text)}  ({reply.seconds:.0f}s{_cost(reply)})")

        tail = reply.text.rstrip()[-40:]
        if CONVERGED in tail or IMPASSE in tail:
            stopped = CONVERGED if CONVERGED in tail else IMPASSE
            log(f"\n   {speaker.name} called {stopped} - ending the discussion early")
            break

    transcript.note = stopped
    if synthesizer != "none":
        speaker = a if (a.name.lower() == synthesizer.lower() or a.kind == synthesizer) else b
        log(f"\n>> Synthesis by {speaker.name}")
        transcript.synthesis = await ask(
            speaker,
            synthesis_prompt(question, transcript.render(include_synthesis=False), synthesizer=speaker.name),
            cwd=repo, scratch=scratch, timeout=timeout, fake=fake,
        )
        log(f"   done ({transcript.synthesis.seconds:.0f}s{_cost(transcript.synthesis)})")
    return transcript


def build_comparison(transcript: Transcript) -> str:
    """Both speakers' structured blocks side by side - the point of --schema.

    Two prose answers are hard to compare; two blocks with the same keys are not.
    Confidence movement across the exchange is the most informative part: a
    speaker who did not budge at all is as interesting as one who flipped.
    """
    spec = transcript.spec or {}
    have = [(stage, r) for stage, r in transcript.turns if r.payload]
    if not have:
        return "# Structured comparison\n\n_No valid structured blocks were produced._\n"

    speakers: list[str] = []
    for _, reply in have:
        if reply.speaker not in speakers:
            speakers.append(reply.speaker)
    stages: list[str] = []
    for stage, _ in have:
        if stage not in stages:
            stages.append(stage)

    def cell(stage: str, speaker: str, field_name: str) -> Any:
        for s, reply in have:
            if s == stage and reply.speaker == speaker:
                return (reply.payload or {}).get(field_name)
        return None

    out = [f"# Structured comparison", "", f"**Question:** {transcript.question}", ""]

    scalars = [n for n, k in spec.items() if k in ("str", "number", "confidence")]
    for field_name in scalars:
        out += [f"## {field_name}", "", "| Stage | " + " | ".join(speakers) + " |",
                "| --- | " + " | ".join("---" for _ in speakers) + " |"]
        for stage in stages:
            values = []
            for speaker in speakers:
                value = cell(stage, speaker, field_name)
                if isinstance(value, float):
                    values.append(f"{value:.2f}")
                else:
                    values.append(str(value or "-").replace("|", "\\|").replace("\n", " "))
            out.append(f"| {stage} | " + " | ".join(values) + " |")
        out.append("")

    confidence_fields = [n for n, k in spec.items() if k == "confidence"]
    if confidence_fields and len(stages) > 1:
        field_name = confidence_fields[0]
        out += [f"## Movement in {field_name}", ""]
        for speaker in speakers:
            first = cell(stages[0], speaker, field_name)
            last = cell(stages[-1], speaker, field_name)
            if isinstance(first, (int, float)) and isinstance(last, (int, float)):
                delta = float(last) - float(first)
                arrow = "unchanged" if abs(delta) < 1e-9 else f"{delta:+.2f}"
                out.append(f"- **{speaker}**: {float(first):.2f} -> {float(last):.2f} ({arrow})")
        out += ["", "_A speaker who did not move at all is worth a second look: either the "
                    "other side had nothing, or they were not listening._", ""]

    lists = [n for n, k in spec.items() if k == "list[str]"]
    for field_name in lists:
        out += [f"## {field_name}", ""]
        for speaker in speakers:
            out.append(f"**{speaker}**")
            out.append("")
            for stage in stages:
                items = cell(stage, speaker, field_name) or []
                if not isinstance(items, list) or not items:
                    continue
                out.append(f"_{stage}_")
                out += [f"- {item}" for item in items]
                out.append("")
    return "\n".join(out) + "\n"


def _preview(text: str, width: int = 66) -> str:
    """First meaningful line of a turn, for live progress."""
    for line in text.strip().splitlines():
        clean = line.strip().lstrip("#*_- ").strip()
        if clean:
            return clean[:width] + ("..." if len(clean) > width else "")
    return "(empty)"


async def converse(
    a: Speaker,
    b: Speaker,
    question: str,
    *,
    rounds: int,
    synthesizer: str,
    repo: Path,
    scratch: Path,
    timeout: int,
    fake: bool,
    log,
    context: str = "",
    profile: Profile = SOFTWARE,
    spec: dict[str, str] | None = None,
    repairs: int = 1,
    transcript: Transcript | None = None,
) -> Transcript:
    transcript = transcript if transcript is not None else Transcript(question=question, spec=spec)
    transcript.spec = spec
    call = dict(cwd=repo, scratch=scratch, timeout=timeout, fake=fake)
    structured = dict(spec=spec, repairs=repairs, **call)

    log(f"\n>> Independent answers ({a.name} and {b.name}, in parallel)")
    first = await asyncio.gather(
        ask_structured(a, opening_prompt(a, question, repo=repo, other=b.name, spec=spec, context=context, profile=profile), **structured),
        ask_structured(b, opening_prompt(b, question, repo=repo, other=a.name, spec=spec, context=context, profile=profile), **structured),
    )
    latest = {a.name: first[0], b.name: first[1]}
    for reply in first:
        transcript.add("Independent answer", reply)
        log(f"   {reply.speaker} answered ({reply.seconds:.0f}s{_cost(reply)}{_repaired(reply)})")

    for turn in range(1, rounds + 1):
        final = turn == rounds
        log(f"\n>> Exchange {turn} of {rounds}")
        replies = await asyncio.gather(
            ask_structured(
                a,
                exchange_prompt(a, question, repo=repo, other=b.name, other_text=latest[b.name].text,
                                own_text=latest[a.name].text, turn=turn, final_turn=final, spec=spec, context=context, profile=profile),
                **structured,
            ),
            ask_structured(
                b,
                exchange_prompt(b, question, repo=repo, other=a.name, other_text=latest[a.name].text,
                                own_text=latest[b.name].text, turn=turn, final_turn=final, spec=spec, context=context, profile=profile),
                **structured,
            ),
        )
        for reply in replies:
            transcript.add(f"Exchange {turn}", reply)
            log(f"   {reply.speaker} replied ({reply.seconds:.0f}s{_cost(reply)}{_repaired(reply)})")
        latest = {a.name: replies[0], b.name: replies[1]}

    if synthesizer != "none":
        speaker = a if a.name.lower() == synthesizer.lower() or a.kind == synthesizer else b
        log(f"\n>> Synthesis by {speaker.name}")
        transcript.synthesis = await ask(
            speaker,
            synthesis_prompt(question, transcript.render(include_synthesis=False), synthesizer=speaker.name),
            cwd=repo, scratch=scratch, timeout=timeout, fake=fake,
        )
        log(f"   done ({transcript.synthesis.seconds:.0f}s{_cost(transcript.synthesis)})")

    return transcript


def _cost(reply: Reply) -> str:
    return f", ${reply.cost_usd:.4f}" if reply.cost_usd else ""


def _repaired(reply: Reply) -> str:
    if reply.repairs and reply.payload is not None:
        return f", block repaired x{reply.repairs}"
    if reply.repairs and reply.payload is None:
        return ", BLOCK INVALID"
    return ""


# ----------------------------------------------------------------------
# personas
# ----------------------------------------------------------------------

FABLE_PERSONA = """\
Your instinct is to look at the system as a whole: boundaries, failure modes,
what happens under load or when a dependency dies, security and data handling,
and what this will cost to operate and maintain a year from now. You care about
what breaks and who gets paged.

Play to that instinct, but answer the actual question. Do not refuse to comment
on implementation detail if it matters."""

SOL_PERSONA = """\
Your instinct is to look at what it takes to actually build this: how it fits the
code that already exists, where the seams are, the concrete steps in order, what
the migration looks like, and what test or prototype would prove it works. You
care about whether it can be shipped.

Play to that instinct, but answer the actual question. Do not refuse to comment
on operational or design concerns if they matter."""


def load_persona(name: str, default: str, *, enabled: bool = True,
                 profile: str = "software") -> str:
    """Persona text from a file, if present, else the profile's built-in default.

    Files are looked up per profile - `personas/general-fable.md` before
    `personas/fable.md` - because a lens written for code ("where the seams are",
    "the migration") is wrong for a question about a drone. The unprefixed files
    apply to the software profile only, which is what they were written for.

    Everything before the first `---` line is notes for the human editing the
    file and is stripped; only what follows reaches the model. Without that split,
    "edit this file to..." ends up in the prompt as an instruction.
    """
    if not enabled:
        # No lens at all: both get byte-identical prompts, so any difference in
        # the answers is the models rather than something you told them to be.
        return ""
    candidates = [PERSONA_DIR / f"{profile}-{name.lower()}.md"]
    if profile == "software":
        candidates.append(PERSONA_DIR / f"{name.lower()}.md")
    path = next((c for c in candidates if c.is_file()), None)
    if path is None:
        return default
    text = path.read_text(encoding="utf-8")
    parts = re.split(r"^---\s*$", text, maxsplit=1, flags=re.MULTILINE)
    body = (parts[1] if len(parts) > 1 else parts[0]).strip()
    if len(parts) == 1:  # no separator: drop a leading H1 and use the rest
        body = re.sub(r"^#.*$", "", body, count=1, flags=re.MULTILINE).strip()
    return body or default


# ----------------------------------------------------------------------
# fake mode
# ----------------------------------------------------------------------


def _canned_payload(speaker: Speaker, prompt: str) -> str:
    """A schema-shaped block for --fake, built from the template in the prompt."""
    match = re.search(r"```json\n(\{.*?\})\n```", prompt, re.DOTALL)
    if not match:
        return ""
    try:
        template = json.loads(re.sub(r"0\.0|\b0\b", "0", match.group(1)))
    except json.JSONDecodeError:
        return ""
    later = "Exchange" in prompt or "Fix your structured" in prompt
    filled: dict[str, Any] = {}
    for key, value in template.items():
        if isinstance(value, list):
            filled[key] = [f"{speaker.name}: {key} item one", f"{speaker.name}: {key} item two"]
        elif isinstance(value, (int, float)):
            filled[key] = (0.72 if speaker.name == "Fable" else 0.55) + (0.08 if later else 0)
        else:
            filled[key] = f"{speaker.name}: extract behind an adapter first"
    return "\n\n```json\n" + json.dumps(filled, indent=2) + "\n```\n"


def _canned(speaker: Speaker, prompt: str) -> str:
    if prompt.lstrip().startswith("# Discussion"):
        match = re.search(r"turn (\d+) of (\d+)", prompt)
        turn, total = (int(match.group(1)), int(match.group(2))) if match else (1, 1)
        if turn == 1:
            return (
                "I'd keep billing in place behind a module boundary first. The seam is "
                "untested, and extracting it now means discovering the boundary is wrong "
                "while also running two deploys.\n\n_(canned --fake turn; no model called)_"
            )
        if turn == total:
            return (
                "Where I land: module boundary first, extraction once the write path is "
                "traced. You moved me on the deploy risk being smaller than I said. Still "
                "unresolved between us: whether the trace is worth a week.\n\n"
                "_(canned --fake turn; no model called)_"
            )
        return (
            f"Picking up your point about the deploy cost - that's fair, I overstated it. "
            f"But you're treating the write volume as known and neither of us has measured "
            f"it. What would change your mind?\n\n_(canned --fake turn {turn}; no model called)_"
        )
    if prompt.lstrip().startswith("# Synthesis"):
        return (
            "**Short answer.** Both of us favour an incremental path; we disagree on "
            "whether the shadow-read phase is worth its complexity.\n\n"
            "**What we agreed on.** The seam belongs at the billing module boundary.\n\n"
            "**What we did not agree on.** Sol thinks shadow reads are premature; I "
            "think cutover without them is unsafe. A one-hour production trace of "
            "concurrent writers would settle it.\n\n"
            "**Assumptions this rests on.** That peak write volume is near 400 rps - "
            "neither of us verified this.\n\n"
            "**My recommendation.** Incremental extraction, with shadow reads kept "
            "until the trace comes back. I'd change my mind if the trace shows a "
            "single writer.\n\n_(canned text from --fake; no model was called)_"
        )
    if prompt.lstrip().startswith("# Exchange"):
        return (
            f"_{speaker.name} responding to the other advisor._\n\n"
            "**Where they're right.** The single-writer assumption is load-bearing and "
            "I did not check it. Conceded.\n\n"
            "**Where I disagree.** They treat 400 rps as a design input; it is an "
            "estimate nobody measured.\n\n"
            "**What would settle it.** One hour of production write traces.\n\n"
            "_(canned text from --fake; no model was called)_"
        )
    return (
        f"_{speaker.name}'s independent answer._\n\n"
        "I'd extract the module behind an adapter rather than standing up a new "
        "service immediately, because the boundary is untested.\n\n"
        "**Assuming** peak write volume is around 400 rps - I have not verified "
        "this - a single-writer cutover is the main risk.\n\n"
        "_(canned text from --fake; no model was called)_"
    )


# ----------------------------------------------------------------------
# cli
# ----------------------------------------------------------------------


def find_codex(explicit: str | None) -> str:
    if explicit:
        return explicit
    if shutil.which("codex"):
        return "codex"
    for pattern in CODEX_CANDIDATES:
        for match in sorted(glob.glob(os.path.expanduser(pattern)), reverse=True):
            if os.path.isfile(match):
                return match
    return "codex"  # let resolve() produce the helpful error


def slugify(text: str, limit: int = 40) -> str:
    slug = re.sub(r"[^a-z0-9]+", "-", text.lower()).strip("-")
    return slug[:limit].rstrip("-") or "question"


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="duet",
        description=(
            "Have Claude Code and Codex answer a question independently, critique each "
            "other, and produce a synthesis. Uses your Claude and ChatGPT "
            "subscriptions via the local CLIs - no API keys."
        ),
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""examples:
  duet.py --fake "should we split billing out?"        see the flow, zero calls
  duet.py --dry-run "..."                              show the commands, run nothing
  duet.py "is this cache invalidation correct?"         5 calls: 2 + 2 + 1
  duet.py --mode dialogue "monolith or services?"       real turn-taking, 6 turns
  duet.py --mode dialogue --turns 4 --first sol "..."   Sol opens, 4 turns
  duet.py --rounds 2 --synthesizer none "..."           longer debate, no synthesis
  duet.py --repo ~/code/api "where should auth live?"   let them read a codebase

two modes:
  parallel   both answer blind, then critique each other. Independent, but they
             never actually hear each other mid-thought.
  dialogue   strict alternating turns, each speaker sees the real history. Slower
             (sequential, not parallel) but it is an actual conversation.
""",
    )
    parser.add_argument("question", nargs="?", help="the question, or - to read stdin")
    parser.add_argument("--mode", default="parallel", choices=["parallel", "dialogue"],
                        help="parallel: independent answers then critiques. "
                             "dialogue: sequential turn-taking (default: parallel)")
    parser.add_argument("--turns", type=int, default=6, metavar="N",
                        help="dialogue mode: total alternating turns (default: 6, i.e. 3 each)")
    parser.add_argument("--first", default="fable", choices=["fable", "sol"],
                        help="dialogue mode: who opens (default: fable)")
    parser.add_argument("--rounds", type=int, default=1, metavar="N",
                        help="parallel mode: critique exchanges after the independent answers (default: 1)")
    # A flag plus a separate file option, deliberately: `--schema` with an optional
    # value would swallow the question in `duet --schema "my question"`.
    # default=None means "not specified", which lets dialogue mode opt out silently
    # while still erroring if you ask for a schema there explicitly.
    parser.add_argument("--schema", action=argparse.BooleanOptionalAction, default=None,
                        help="require a structured JSON block from each answer. "
                             "On by default in parallel mode; use --no-schema for prose only")
    parser.add_argument("--schema-file", metavar="FILE",
                        help="use your own schema instead of the built-in one: a JSON object of "
                             "field -> type name or example value (implies --schema)")
    parser.add_argument("--schema-repairs", type=int, default=1, metavar="N",
                        help="retries allowed when a structured block fails validation (default: 1)")
    parser.add_argument("--synthesizer", default="claude", choices=["claude", "codex", "none"],
                        help="who writes the final answer; 'none' prints both sides raw (default: claude)")
    parser.add_argument("--profile", default="software", choices=sorted(PROFILES),
                        help="software: cites files, hands over a repo, code-shaped lenses. "
                             "general: no repo, no code vocabulary - use this for anything "
                             "that is not a codebase (default: software)")
    parser.add_argument("--no-personas", action="store_true",
                        help="drop both lenses entirely, so the two get identical prompts")
    parser.add_argument("--repo", default=".", help="directory both may read, read-only (default: cwd)")
    parser.add_argument("--context", action="append", metavar="FILE",
                        help="file whose contents go into BOTH prompts verbatim. Repeatable. "
                             "May be a quoted glob: --context 'docs/*.md'. "
                             f"{CONVENTION_FILE} in --repo is included automatically")
    parser.add_argument("--add-dir", action="append", metavar="DIR",
                        help="extra directory both may read. Repeatable")
    parser.add_argument("--out", help=f"transcript directory (default: {DEFAULT_OUT})")
    parser.add_argument("--claude-bin", default="claude", help="path to the claude CLI")
    parser.add_argument("--codex-bin", help="path to the codex CLI (auto-detected)")
    # Both default to empty: inherit whatever each CLI is already configured with
    # (~/.claude/settings.json, ~/.codex/config.toml). Forcing a model here would
    # silently override a deliberate choice like "opus[1m]".
    parser.add_argument("--claude-model", default="",
                        help="model for Claude Code, e.g. fable, opus, sonnet, haiku, "
                             "claude-fable-5, 'opus[1m]' (default: your Claude Code setting)")
    parser.add_argument("--codex-model", default="",
                        help="model for Codex, e.g. gpt-5.6-sol (default: your ~/.codex/config.toml)")
    parser.add_argument("--claude-effort", default="", metavar="LEVEL",
                        choices=["", "low", "medium", "high", "xhigh", "max"],
                        help="reasoning effort for Claude Code: low, medium, high, xhigh, max "
                             "(default: your Claude Code setting)")
    parser.add_argument("--codex-effort", default="", metavar="LEVEL",
                        help="reasoning effort for Codex, e.g. low, medium, high "
                             "(default: your ~/.codex/config.toml)")
    parser.add_argument("--timeout", type=int, default=900, help="per-call timeout in seconds")
    parser.add_argument("--quiet", action="store_true", help="only print the final answer")
    parser.add_argument("--dry-run", action="store_true", help="print the commands and prompts, invoke nothing")
    parser.add_argument("--fake", action="store_true", help="run the whole flow with canned answers, zero calls")
    parser.add_argument("--status", action="store_true",
                        help="report where transcripts live, how many there are and their size, then exit")
    return parser


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)

    if args.status:
        out_dir = Path(args.out).expanduser() if args.out else DEFAULT_OUT
        print(runs_summary(out_dir, with_size=True))
        try:
            import sessions

            rows = sessions.SessionStore().list()
            print(f"{len(rows)} panel conversation(s) in {sessions.sessions_dir()}")
            for row in rows[:8]:
                print(f"  {row['cid'][:8]}  gen {row['generation']}  "
                      f"{row['exchanges']} exchange(s), {row['in_context']} in context  "
                      f"{row['title'][:44]}")
        except Exception as exc:  # the CLI must not depend on the panel's store
            print(f"(conversation store unavailable: {exc})")
        warning = sync_warning(out_dir)
        print(f"WARNING: {warning}" if warning else "not in a cloud-synced folder")
        return 0

    question = args.question
    if question == "-" or (question is None and not sys.stdin.isatty()):
        question = sys.stdin.read()
    if not question or not question.strip():
        return _fail("no question given. Pass it as an argument, or use - to read stdin.")
    question = question.strip()

    repo = Path(args.repo).expanduser().resolve()
    if not repo.is_dir():
        return _fail(f"--repo is not a directory: {repo}")
    if args.rounds < 0:
        return _fail("--rounds cannot be negative")
    if args.turns < 1:
        return _fail("--turns must be at least 1")

    try:
        ctx = load_context(args.context, args.add_dir, repo)
    except DuetError as exc:
        return _fail(str(exc))
    add_dirs = tuple(ctx.add_dirs)

    profile = PROFILES[args.profile]
    personas_on = not args.no_personas
    fable = Speaker("Fable", "claude", args.claude_bin, args.claude_model,
                    load_persona("fable", profile.fable_persona, enabled=personas_on,
                                 profile=profile.name),
                    args.claude_effort, add_dirs)
    sol = Speaker("Sol", "codex", find_codex(args.codex_bin), args.codex_model,
                  load_persona("sol", profile.sol_persona, enabled=personas_on,
                               profile=profile.name),
                  args.codex_effort, add_dirs)

    dialogue = args.mode == "dialogue"

    asked_for_schema = args.schema is True or bool(args.schema_file)
    if dialogue and asked_for_schema:
        return _fail(
            "--schema is a parallel-mode feature.\n"
            "  Dialogue turns are meant to be a few sentences; demanding a full block\n"
            "  every turn fights that and bloats the history. Use --mode parallel for a\n"
            "  structured comparison, or drop --schema for a discussion."
        )
    # On by default in parallel mode, off in dialogue mode, unless overridden.
    schema_on = args.schema if args.schema is not None else not dialogue

    spec: dict[str, str] | None = None
    if schema_on and not dialogue:
        try:
            spec = load_schema(args.schema_file)
        except DuetError as exc:
            return _fail(str(exc))
    if args.schema_repairs < 0:
        return _fail("--schema-repairs cannot be negative")

    synth_calls = 0 if args.synthesizer == "none" else 1
    calls = (args.turns if dialogue else 2 + 2 * args.rounds) + synth_calls
    log = (lambda *_: None) if args.quiet else (lambda msg: print(msg, flush=True))
    opener, second = (fable, sol) if args.first == "fable" else (sol, fable)

    if args.dry_run:
        return _dry_run(opener, second, question, repo=repo, calls=calls,
                        dialogue=dialogue, turns=args.turns, spec=spec, context=ctx.block(profile),
                        profile=profile)

    if not args.fake:
        try:
            for speaker in (fable, sol):
                speaker.resolve()
        except DuetError as exc:
            return _fail(str(exc))

    synth_note = f" + synthesis by {args.synthesizer}" if args.synthesizer != "none" else " + no synthesis"

    log(f"duet: {fable.label} and {sol.label}")
    log(f"  question: {question if len(question) < 90 else question[:87] + '...'}")
    log(f"  profile:  {profile.name}"
        + ("" if personas_on else "   personas: off (identical prompts)"))
    if profile.show_repo:
        log(f"  reading:  {repo}")
    if ctx.files:
        log(f"  context:  {len(ctx.files)} file(s), {ctx.chars:,} chars, identical for both")
        for label, body in ctx.files:
            log(f"              {label} ({len(body):,})")
    elif args.context:
        log("  context:  none loaded")
    for note in ctx.warnings:
        log(f"  WARNING:  {note}")
    if ctx.add_dirs:
        log(f"  add-dir:  {', '.join(str(d) for d in ctx.add_dirs)}")
    warning = context_asymmetry(repo)
    if warning:
        log(f"  WARNING:  {warning}")
    if ctx.chars > CONTEXT_WARN_CHARS:
        log(f"  WARNING:  context is large; it is resent on every call ({calls} of them)")
    for speaker in (fable, sol):
        log(f"  {speaker.name + ':':<9} model={speaker.model or 'from your ' + speaker.kind + ' config'}"
            f"  effort={speaker.effort or 'from your ' + speaker.kind + ' config'}")
    if dialogue:
        log(f"  plan:     dialogue, {args.turns} alternating turns, {opener.name} opens"
            + synth_note + f" = {calls} call(s), sequential")
    else:
        log(f"  plan:     2 independent answers + {args.rounds} exchange(s)"
            + synth_note + f" = {calls} call(s)")
    if args.fake:
        log("  MODE:     --fake, canned answers, nothing is invoked")

    stamp = datetime.now().strftime("%Y%m%d-%H%M%S")
    out_dir = Path(args.out).expanduser() if args.out else DEFAULT_OUT
    run_dir = out_dir / f"{stamp}-{slugify(question)}"
    log(f"  writing:  {run_dir}")
    synced = sync_warning(out_dir)
    if synced:
        log(f"  WARNING:  {synced}")

    shared = dict(
        context=ctx.block(profile),
        profile=profile,
        synthesizer=args.synthesizer,
        repo=repo,
        scratch=run_dir / ".scratch",
        timeout=args.timeout,
        fake=args.fake,
        log=log,
    )
    # Owned here, not inside the loops, so an interrupted run still has whatever
    # was completed before it died.
    transcript = Transcript(question=question, spec=spec)
    shared["transcript"] = transcript
    try:
        if dialogue:
            coro = discuss(opener, second, question, turns=args.turns, **shared)
        else:
            coro = converse(fable, sol, question, rounds=args.rounds,
                            spec=spec, repairs=args.schema_repairs, **shared)
        asyncio.run(coro)
    except (DuetError, KeyboardInterrupt) as exc:
        interrupted = isinstance(exc, KeyboardInterrupt)
        reason = "interrupted" if interrupted else str(exc)
        if transcript.turns:
            done = len(transcript.turns)
            transcript.note = f"INCOMPLETE - {reason} after {done} completed turn(s)"
            persist(transcript, run_dir, spec=spec)
            shutil.rmtree(run_dir / ".scratch", ignore_errors=True)
            print(f"duet: {reason}. Kept {done} completed turn(s): "
                  f"{run_dir / 'transcript.md'}", file=sys.stderr)
            return 130 if interrupted else 2
        return _fail(reason, code=130 if interrupted else 2)

    body = persist(transcript, run_dir, spec=spec)
    shutil.rmtree(run_dir / ".scratch", ignore_errors=True)

    if args.quiet:
        print((transcript.synthesis or transcript.turns[-1][1]).text)
    else:
        print("\n" + "=" * 72)
        if transcript.synthesis is not None:
            print(f"SYNTHESIS by {transcript.synthesis.speaker}")
            print("=" * 72 + "\n")
            print(transcript.synthesis.text)
        else:
            print("NO SYNTHESIS - both sides, unreconciled")
            print("=" * 72 + "\n")
            print(transcript.render())
        if spec is not None:
            print("\n" + "=" * 72)
            print("STRUCTURED COMPARISON")
            print("=" * 72 + "\n")
            print(build_comparison(transcript))
        print("\n" + "-" * 72)
        if transcript.note:
            print(f"ended early:  a speaker called {transcript.note}")
        print(f"transcript: {run_dir / 'transcript.md'}")
        print(f"runs:       {runs_summary(out_dir)}")
        if transcript.total_cost:
            note = " (notional - subscription plans are not billed per call)"
            print(f"reported:   ${transcript.total_cost:.4f}{note}")
    return 0


def _dry_run(
    first: Speaker,
    second: Speaker,
    question: str,
    *,
    repo: Path,
    calls: int,
    dialogue: bool,
    turns: int,
    spec: dict[str, str] | None = None,
    context: str = "",
    profile: Profile = SOFTWARE,
) -> int:
    print("DRY RUN - nothing will be invoked.\n")
    print(f"Profile: {profile.name}\n")
    if spec:
        print(f"Schema on: {', '.join(spec)}\n")
    if dialogue:
        print(f"Mode: dialogue. {turns} alternating turns, run one at a time, "
              f"{first.name} opens.")
    else:
        print("Mode: parallel. Both answer at once, then critique each other.")
    print(f"Would make {calls} call(s) total.\n")

    def first_prompt(speaker: Speaker, other: Speaker) -> str:
        if dialogue:
            return dialogue_prompt(speaker, question, repo=repo, other=other.name,
                                   history=[], turn=1, total=turns, context=context, profile=profile)
        return opening_prompt(speaker, question, repo=repo, other=other.name, spec=spec,
                              context=context, profile=profile)

    for speaker, other in ((first, second), (second, first)):
        try:
            argv, _, _ = speaker.command("PROMPT", cwd=repo, scratch=Path("/tmp"))
            shown = " ".join(argv)
        except DuetError as exc:
            shown = f"(cannot resolve: {exc.args[0].splitlines()[0]})"
        prompt = first_prompt(speaker, other)
        print(f"{speaker.label}")
        print(f"  {shown}")
        print(f"  first prompt: {len(prompt):,} chars on stdin (~{len(prompt) // 4:,} tokens)")
        if dialogue:
            print("  later turns grow as the history accumulates")
        print()

    print(f"{'Opening' if dialogue else 'Independent-answer'} prompt for {first.name}:\n")
    print("-" * 72)
    print(first_prompt(first, second))
    print("-" * 72)
    print("\nNothing was invoked. Drop --dry-run to run it.")
    return 0


def persist(transcript: Transcript, run_dir: Path, *, spec: dict[str, str] | None) -> str:
    """Write everything we have. Called on success *and* on interruption.

    A run that dies at turn four used to write nothing at all: the mkdir and the
    writes only happened after the whole coroutine returned, so a timeout, a
    DuetError or a Ctrl-C threw away every turn already paid for. One shared path
    means the partial case cannot drift from the complete one.
    """
    run_dir.mkdir(parents=True, exist_ok=True)
    body = transcript.render()
    (run_dir / "transcript.md").write_text(body, encoding="utf-8")
    for index, (stage, reply) in enumerate(transcript.turns, start=1):
        stem = f"{index:02d}-{slugify(stage)}-{reply.speaker.lower()}"
        (run_dir / f"{stem}.md").write_text(reply.text + "\n", encoding="utf-8")
        if reply.payload is not None:
            (run_dir / f"{stem}.json").write_text(
                json.dumps(reply.payload, indent=2) + "\n", encoding="utf-8"
            )
    if spec is not None:
        (run_dir / "comparison.md").write_text(build_comparison(transcript), encoding="utf-8")
    if transcript.synthesis is not None:
        (run_dir / "synthesis.md").write_text(
            transcript.synthesis.text + "\n", encoding="utf-8")
    return body


def runs_summary(out_dir: Path, *, with_size: bool = False) -> str:
    """How much has piled up. Count is cheap; size walks the tree, so it is opt-in."""
    if not out_dir.is_dir():
        return "no runs yet"
    dirs = [d for d in out_dir.iterdir() if d.is_dir()]
    text = f"{len(dirs)} run(s) in {out_dir}"
    if with_size:
        total = sum(f.stat().st_size for d in dirs for f in d.rglob("*") if f.is_file())
        text += f", {total / 1024:.0f} KB" if total < 1024 * 1024 else f", {total / 1048576:.1f} MB"
    return text


def _fail(message: str, code: int = 2) -> int:
    print(f"duet: {message}", file=sys.stderr)
    return code


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""duet serve - JSONL backend for the VS Code extension.

The extension is a thin UI. All the substance - prompts, personas, context,
schema, the two CLIs, streaming - stays here in Python, already tested by
chat.py's pty suite. Reimplementing any of it in JavaScript would mean two
versions to keep honest.

Protocol: one JSON object per line, both directions.

in:   {"id": 1, "cmd": "ask", "question": "...", "settings": {...}}
      {"cmd": "cancel"}
      {"cmd": "ping"}
      {"cmd": "probe"}                      report which CLIs are available
      {"cmd": "attach",  "conversationId": "..."}   rehydrate a tab after reload
      {"cmd": "forget",  "conversationId": "..."}   delete a conversation for good
      {"cmd": "reset",   "conversationId": "..."}   new generation, keeps the record
      {"cmd": "sessions"}                   list stored conversations

out:  {"event": "ready",  "codex": "...", "claude": "..."}
      {"id": 1, "event": "start",   "calls": 5, "plan": "..."}
      {"id": 1, "event": "stage",   "stage": "Independent answer", "status": "..."}
      {"id": 1, "event": "delta",   "speaker": "Fable", "text": "..."}
      {"id": 1, "event": "message", "speaker": "Fable", "stage": "...",
                "text": "...", "items": [...], "cost": 0.18, "model": "...",
                "tokens": {...}, "narration": [...]}
      {"id": 1, "event": "done",    "calls": 5, "cost": 0.31, "seconds": 120}
      {"id": 1, "event": "error",   "message": "..."}
      {"event": "attached",  "conversationId": "...", "restored": true,
                             "generation": 2, "exchanges": [...], "inContext": 1}
      {"event": "reset",     "conversationId": "...", "generation": 3}
      {"event": "forgotten", "conversationId": "..."}
      {"event": "sessions",  "sessions": [...], "dir": "..."}

stdout carries protocol only. Diagnostics go to stderr.
"""

from __future__ import annotations

import asyncio
import json
import queue
import shutil
import sys
import threading
import time
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any

import chat
import duet
import sessions

_LOCK = threading.Lock()


def emit(payload: dict[str, Any]) -> None:
    with _LOCK:
        sys.stdout.write(json.dumps(payload, ensure_ascii=False) + "\n")
        sys.stdout.flush()


def log(message: str) -> None:
    sys.stderr.write(f"[duet serve] {message}\n")
    sys.stderr.flush()


# ----------------------------------------------------------------------
# settings coming from the extension
# ----------------------------------------------------------------------


def _int(raw: dict[str, Any], key: str, default: int, *, low: int) -> int:
    """Read an int, defaulting only when it is actually absent.

    `int(raw.get("rounds") or 1)` looks harmless and silently turns an explicit 0
    into 1, so asking for no critique round bought two extra calls.
    """
    value = raw.get(key)
    if value is None or isinstance(value, bool):
        return default
    try:
        return max(low, int(value))
    except (TypeError, ValueError):
        return default


def _repo(raw: dict[str, Any], profile: str) -> Path:
    """The directory both speakers may read.

    Defaulting to the working directory reads as harmless and is not: the backend's
    working directory is wherever serve.py lives, so with no folder open in VS Code
    the software profile handed both CLIs duet's own source instead of the user's
    code, and said nothing. The extension refuses that case before sending; this is
    the half a client cannot talk its way past.

    General profile has no repository by design, and never puts repository content
    in a prompt, so falling back there is fine - it only decides which directory the
    two read-only CLIs are started in.
    """
    given = str(raw.get("repo") or "").strip()
    if not given:
        if profile == "software":
            raise duet.DuetError(
                "no repository to read: open a folder in VS Code, or switch to the general profile"
            )
        return Path.cwd()
    repo = Path(given).expanduser()
    if not repo.is_dir():
        raise duet.DuetError(f"not a directory: {repo}")
    return repo.resolve()


def build_settings(raw: dict[str, Any]) -> chat.Settings:
    profile = str(raw.get("profile") or "software")
    return chat.Settings(
        mode=str(raw.get("mode") or "parallel"),
        profile=profile,
        personas=raw.get("personas") is not False,
        rounds=_int(raw, "rounds", 1, low=0),
        turns=_int(raw, "turns", 6, low=1),
        first=str(raw.get("first") or "fable"),
        synthesizer=str(raw.get("synthesizer") or "none"),
        schema=bool(raw.get("schema")),
        repo=_repo(raw, profile),
        context_patterns=list(raw.get("context") or []),
        claude_model=str(raw.get("claudeModel") or ""),
        claude_effort=str(raw.get("claudeEffort") or ""),
        codex_model=str(raw.get("codexModel") or ""),
        codex_effort=str(raw.get("codexEffort") or ""),
        timeout=_int(raw, "timeout", 900, low=30),
    )


def make_speakers(settings: chat.Settings, claude_bin: str, codex_bin: str) -> tuple[duet.Speaker, duet.Speaker]:
    prof = duet.PROFILES.get(settings.profile, duet.SOFTWARE)
    fable = duet.Speaker(
        "Fable", "claude", claude_bin or "claude", settings.claude_model,
        duet.load_persona("fable", prof.fable_persona,
                          enabled=settings.personas, profile=prof.name),
        settings.claude_effort,
    )
    sol = duet.Speaker(
        "Sol", "codex", codex_bin or duet.find_codex(None), settings.codex_model,
        duet.load_persona("sol", prof.sol_persona,
                          enabled=settings.personas, profile=prof.name),
        settings.codex_effort,
    )
    return fable, sol


# ----------------------------------------------------------------------
# one ask
# ----------------------------------------------------------------------


@dataclass
class Conversation:
    """One tab. Everything here used to live on Runner and leak between tabs.

    The history leak was the visible one - a steak discussion feeding into the
    next question's context - but the cancel Event was worse: one tab's stop
    button killed another tab's in-flight run. Cost and call counts were shared
    too, so every tab showed the same cumulative spend.
    """

    cid: str
    cancel: threading.Event = field(default_factory=threading.Event)
    cost: float = 0.0
    calls: int = 0
    # The persisted record is the source of truth for what may enter a prompt.
    # Keeping it here rather than a separate in-memory list means the two cannot
    # drift, which is the bug class that makes "reset cleared it" untrue.
    record: dict[str, Any] = field(default_factory=dict)


class Runner:
    def __init__(self, *, fake: bool, claude_bin: str, codex_bin: str) -> None:
        self.fake = fake
        self.claude_bin = claude_bin
        self.codex_bin = codex_bin
        self.conversations: dict[str, Conversation] = {}
        self.store = sessions.SessionStore()
        # Guards active_id / active_cid / cancelled together. The reader thread and
        # the dispatch loop both touch them, and the ordering is what stops
        # cancel.clear() from swallowing a cancel that arrived while queued.
        self.state = threading.Lock()
        self.active_id: Any = None
        self.active_cid: str | None = None
        self.cancelled: set[Any] = set()

    def conversation(self, cid: str) -> Conversation:
        conv = self.conversations.get(cid)
        if conv is None:
            conv = Conversation(cid=cid, record=self.store.load(cid))
            self.conversations[cid] = conv
        return conv

    def is_active(self, cid: str) -> bool:
        with self.state:
            return self.active_cid == cid

    # ---- context ----

    def context_for(self, conv: Conversation, settings: chat.Settings) -> str:
        blocks = []
        try:
            ctx = duet.load_context(settings.context_patterns, None, settings.repo)
            if ctx.files:
                blocks.append(ctx.block(duet.PROFILES.get(settings.profile, duet.SOFTWARE)))
            for note in ctx.warnings:
                emit({"event": "warning", "message": note})
        except duet.DuetError as exc:
            emit({"event": "warning", "message": str(exc).splitlines()[0]})
        # Current generation only, and nothing the user cancelled - see
        # sessions.prompt_exchanges for why both exclusions exist.
        recent = sessions.prompt_exchanges(conv.record)
        if recent:
            parts = ["## Earlier in this conversation", "",
                     "The human asked these before; you were both present.", ""]
            for exchange in recent:
                parts += [f"**Human:** {exchange['question']}", ""]
                for who, text in exchange["replies"]:
                    body = text if len(text) < 1200 else text[:1200] + " [...]"
                    parts += [f"**{who}:** {body}", ""]
            blocks.append("\n".join(parts))
        return "\n\n".join(blocks)

    # ---- entry ----

    async def ask(self, conv: Conversation, request_id: Any, question: str,
                  settings: chat.Settings) -> None:
        # cancel.clear() happens under self.state in dispatch(), before active_id is
        # published, so a cancel that arrived while this ask was still queued is
        # caught there rather than wiped here.
        fable, sol = make_speakers(settings, self.claude_bin, self.codex_bin)
        if not self.fake:
            for speaker in (fable, sol):
                try:
                    speaker.resolve()
                except duet.DuetError as exc:
                    self.emit_for(conv, {"id": request_id, "event": "error",
                                         "message": str(exc)})
                    return

        calls = (settings.turns if settings.mode == "dialogue" else 2 + 2 * settings.rounds)
        calls += 0 if settings.synthesizer == "none" else 1
        plan = (f"dialogue, {settings.turns} turns, {settings.first} opens"
                if settings.mode == "dialogue"
                else f"parallel, {settings.rounds} exchange(s)")
        self.emit_for(conv, {"id": request_id, "event": "start", "calls": calls,
              "plan": plan, "profile": settings.profile, "personas": settings.personas,
              "fable": {"model": fable.model or "config", "effort": fable.effort or "config"},
              "sol": {"model": sol.model or "config", "effort": sol.effort or "config"}})

        started = time.monotonic()
        collected: list[tuple[str, str]] = []
        context = self.context_for(conv, settings)
        try:
            if settings.mode == "dialogue":
                await self.dialogue(conv, request_id, question, settings, fable, sol, context, collected)
            else:
                await self.parallel(conv, request_id, question, settings, fable, sol, context, collected)
        except duet.DuetError as exc:
            self.emit_for(conv, {"id": request_id, "event": "error", "message": str(exc)})
        except Exception as exc:  # never take the server down with one bad ask
            log(f"ask failed: {type(exc).__name__}: {exc}")
            self.emit_for(conv, {"id": request_id, "event": "error",
                                 "message": f"{type(exc).__name__}: {exc}"})

        if collected:
            # Write through immediately: persisting only at the end of a run leaves
            # a crash window between mutating state and reaching disk. Interrupted
            # runs are stored and flagged, not dropped, so the visible record still
            # matches what the user saw before pressing Stop.
            conv.record = self.store.append(
                conv.cid, question, collected, interrupted=conv.cancel.is_set())
        self.emit_for(conv, {"id": request_id, "event": "done",
              "calls": conv.calls, "cost": round(conv.cost, 6),
              "seconds": round(time.monotonic() - started, 1),
              "interrupted": conv.cancel.is_set()})

    def emit_for(self, conv: Conversation, payload: dict[str, Any]) -> None:
        """Every message carries its conversation, so a tab can ignore other tabs."""
        emit({**payload, "conversationId": conv.cid})

    # ---- shared ----

    def report(self, conv: Conversation, request_id: Any, speaker: str, stage: str,
               res: chat.StreamResult, collected: list[tuple[str, str]]) -> bool:
        conv.calls += 1
        if res.error:
            self.emit_for(conv, {"id": request_id, "event": "error",
                                 "message": f"{speaker}: {res.error}", "speaker": speaker})
            return False
        if res.cost_usd:
            conv.cost += res.cost_usd
        narration = res.items[:-1] if len(res.items) > 1 else []
        body = res.items[-1] if res.items else res.text
        self.emit_for(conv, {"id": request_id, "event": "message", "speaker": speaker, "stage": stage,
              "text": body, "narration": narration, "full": res.text,
              "cost": res.cost_usd, "model": res.model, "tokens": res.tokens})
        collected.append((speaker, res.text))
        return True

    async def run_one(self, conv: Conversation, speaker: duet.Speaker, prompt: str,
                      settings: chat.Settings, *, request_id: Any,
                      stream: bool) -> chat.StreamResult:
        def on_delta(text: str) -> None:
            if stream:
                self.emit_for(conv, {"id": request_id, "event": "delta",
                                     "speaker": speaker.name, "text": text})

        return await chat.stream_speaker(
            speaker, prompt, repo=settings.repo, timeout=settings.timeout,
            fake=self.fake, on_delta=on_delta, cancel=conv.cancel,
        )

    # ---- modes ----

    async def parallel(self, conv: Conversation, request_id: Any, question: str, settings: chat.Settings,
                       fable: duet.Speaker, sol: duet.Speaker, context: str,
                       collected: list[tuple[str, str]]) -> None:
        spec = duet.DEFAULT_SCHEMA if settings.schema else None
        prof = duet.PROFILES.get(settings.profile, duet.SOFTWARE)
        self.emit_for(conv, {"id": request_id, "event": "stage", "stage": "Independent answer",
              "status": "both answering independently"})

        results: dict[str, chat.StreamResult] = {}

        async def one(speaker: duet.Speaker, other: str) -> None:
            prompt = duet.opening_prompt(speaker, question, repo=settings.repo,
                                         other=other, spec=spec, context=context,
                                         profile=prof)
            # Both stream: the UI keeps a separate bubble per speaker, so two
            # simultaneous streams do not collide the way a terminal would.
            results[speaker.name] = await self.run_one(
                conv, speaker, prompt, settings, request_id=request_id, stream=True)

        await asyncio.gather(one(fable, "Sol"), one(sol, "Fable"))
        for name in ("Fable", "Sol"):
            if name in results:
                self.report(conv, request_id, name, "Independent answer", results[name], collected)
        if conv.cancel.is_set():
            return

        latest = {name: res.text for name, res in results.items()}
        for turn in range(1, settings.rounds + 1):
            self.emit_for(conv, {"id": request_id, "event": "stage", "stage": f"Exchange {turn}",
                  "status": f"exchange {turn} of {settings.rounds}"})
            results = {}

            async def one_ex(speaker: duet.Speaker, other: str, turn: int = turn) -> None:
                prompt = duet.exchange_prompt(
                    speaker, question, repo=settings.repo, other=other,
                    other_text=latest.get(other, ""), own_text=latest.get(speaker.name, ""),
                    turn=turn, final_turn=turn == settings.rounds, spec=spec, context=context,
                    profile=prof)
                results[speaker.name] = await self.run_one(
                    conv, speaker, prompt, settings, request_id=request_id, stream=True)

            await asyncio.gather(one_ex(fable, "Sol"), one_ex(sol, "Fable"))
            for name in ("Fable", "Sol"):
                if name in results:
                    self.report(conv, request_id, name, f"Exchange {turn}", results[name], collected)
            if conv.cancel.is_set():
                return
            latest = {name: res.text for name, res in results.items()}

        await self.synthesise(conv, request_id, question, settings, fable, sol, collected)

    async def dialogue(self, conv: Conversation, request_id: Any, question: str, settings: chat.Settings,
                       fable: duet.Speaker, sol: duet.Speaker, context: str,
                       collected: list[tuple[str, str]]) -> None:
        order = [fable, sol] if settings.first == "fable" else [sol, fable]
        history: list[tuple[str, str]] = []

        for turn in range(1, settings.turns + 1):
            if conv.cancel.is_set():
                return
            speaker, other = order[(turn - 1) % 2], order[turn % 2]
            stage = f"Turn {turn}"
            self.emit_for(conv, {"id": request_id, "event": "stage", "stage": stage,
                  "status": f"{speaker.name}, turn {turn} of {settings.turns}",
                  "speaker": speaker.name})
            prompt = duet.dialogue_prompt(
                speaker, question, repo=settings.repo, other=other.name,
                history=history, turn=turn, total=settings.turns, context=context,
                profile=duet.PROFILES.get(settings.profile, duet.SOFTWARE))
            res = await self.run_one(conv, speaker, prompt, settings,
                                     request_id=request_id, stream=True)
            if not self.report(conv, request_id, speaker.name, stage, res, collected):
                return
            history.append((speaker.name, res.text))

            tail = res.text.rstrip()[-40:]
            if duet.CONVERGED in tail or duet.IMPASSE in tail:
                marker = duet.CONVERGED if duet.CONVERGED in tail else duet.IMPASSE
                self.emit_for(conv, {"id": request_id, "event": "stage", "stage": "ended",
                      "status": f"{speaker.name} called {marker}"})
                break

        await self.synthesise(conv, request_id, question, settings, fable, sol, collected)

    async def synthesise(self, conv: Conversation, request_id: Any, question: str, settings: chat.Settings,
                         fable: duet.Speaker, sol: duet.Speaker,
                         collected: list[tuple[str, str]]) -> None:
        if settings.synthesizer == "none" or conv.cancel.is_set() or not collected:
            return
        speaker = fable if settings.synthesizer == "claude" else sol
        transcript = "\n\n".join(f"## {who}\n\n{text}" for who, text in collected)
        self.emit_for(conv, {"id": request_id, "event": "stage", "stage": "Synthesis",
              "status": f"{speaker.name} writing the synthesis", "speaker": speaker.name})
        res = await self.run_one(
            conv, speaker,
            duet.synthesis_prompt(question, transcript, synthesizer=speaker.name),
            settings, request_id=request_id, stream=True)
        self.report(conv, request_id, speaker.name, "Synthesis", res, [])


# ----------------------------------------------------------------------
# loop
# ----------------------------------------------------------------------


def probe(claude_bin: str, codex_bin: str) -> dict[str, Any]:
    def find(name: str, fallback: str) -> str:
        return shutil.which(name) or (str(Path(name).resolve()) if Path(name).is_file() else fallback)

    codex = codex_bin or duet.find_codex(None)
    return {
        "event": "ready",
        "claude": find(claude_bin or "claude", ""),
        "codex": find(codex, ""),
        "cwd": str(Path.cwd()),
        "runsDir": str(duet.DEFAULT_OUT),
        "syncWarning": duet.sync_warning(duet.DEFAULT_OUT) or "",
    }


def main(argv: list[str] | None = None) -> int:
    import argparse

    parser = argparse.ArgumentParser(prog="serve.py", description="JSONL backend for the duet VS Code extension")
    parser.add_argument("--fake", action="store_true", help="canned answers, zero calls")
    parser.add_argument("--claude-bin", default="")
    parser.add_argument("--codex-bin", default="")
    args = parser.parse_args(argv)

    runner = Runner(fake=args.fake, claude_bin=args.claude_bin, codex_bin=args.codex_bin)
    commands: queue.Queue = queue.Queue()
    DEFAULT_CID = "default"

    def cid_of(message: dict[str, Any]) -> str:
        return str(message.get("conversationId") or DEFAULT_CID)

    def reader() -> None:
        """Reads stdin. Cancel and idle reset land here so they do not wait on a run.

        The ordering with dispatch() matters: a cancel for a request that is still
        queued goes into runner.cancelled, which dispatch checks *before* clearing
        the conversation's Event. Handle it the other way round and an early cancel
        is silently swallowed.
        """
        for raw in sys.stdin:
            raw = raw.strip()
            if not raw:
                continue
            try:
                message = json.loads(raw)
            except json.JSONDecodeError:
                emit({"event": "error", "message": f"bad JSON: {raw[:120]}"})
                continue
            cmd = message.get("cmd")
            cid = cid_of(message)

            if cmd == "cancel":
                target = message.get("id")
                with runner.state:
                    if target is not None and target != runner.active_id:
                        # Not running yet: drop it when its turn comes. Keyed by
                        # request id, not conversation, so cancelling one queued ask
                        # does not poison that tab's later requests.
                        runner.cancelled.add(target)
                        emit({"event": "cancelled", "id": target, "conversationId": cid})
                        continue
                    if runner.active_cid == cid:
                        runner.conversation(cid).cancel.set()
                continue

            if cmd == "attach":
                # The client presents a cid; the backend rehydrates from its own
                # file. Deliberately not the reverse - context_for splices history
                # into prompts, so letting client state decide what enters a prompt
                # would let a reloaded tab resurrect exchanges a reset retired.
                conv = runner.conversation(cid)
                record = conv.record or {}
                emit({"event": "attached", "conversationId": cid,
                      "restored": bool(record.get("exchanges")),
                      "generation": record.get("generation", 0),
                      "title": record.get("title", ""),
                      "exchanges": sessions.display_exchanges(record),
                      "inContext": len(sessions.prompt_exchanges(record))})
                continue

            if cmd == "forget":
                # Closing a tab for good, unlike reset which only draws a boundary.
                if not runner.is_active(cid):
                    runner.store.forget(cid)
                    runner.conversations.pop(cid, None)
                    emit({"event": "forgotten", "conversationId": cid})
                    continue
                commands.put(message)
                continue

            if cmd == "sessions":
                emit({"event": "sessions", "sessions": runner.store.list(),
                      "dir": str(runner.store.dir)})
                continue

            if cmd == "reset":
                # An idle tab's reset touches only its own history, which no running
                # ask reads. Make it immediate rather than making the user wait out
                # another tab's ten-minute run.
                if not runner.is_active(cid):
                    conv = runner.conversation(cid)
                    conv.record = runner.store.new_generation(cid)
                    emit({"event": "reset", "conversationId": cid,
                          "generation": conv.record["generation"]})
                    continue
                commands.put(message)   # active: keep it ordered after the run
                continue

            if cmd == "ask":
                depth = commands.qsize()
                if depth or runner.active_id is not None:
                    emit({"event": "queued", "id": message.get("id"),
                          "conversationId": cid, "position": depth + 1})
            commands.put(message)
        commands.put(None)

    threading.Thread(target=reader, daemon=True).start()
    emit(probe(args.claude_bin, args.codex_bin) | {"fake": args.fake})

    while True:
        message = commands.get()
        if message is None:
            return 0
        cmd = message.get("cmd")
        cid = cid_of(message)
        if cmd == "ping":
            emit({"event": "pong"})
        elif cmd == "probe":
            emit(probe(args.claude_bin, args.codex_bin) | {"fake": args.fake})
        elif cmd == "forget":
            runner.store.forget(cid)
            runner.conversations.pop(cid, None)
            emit({"event": "forgotten", "conversationId": cid})
        elif cmd == "reset":
            conv = runner.conversation(cid)
            conv.record = runner.store.new_generation(cid)
            emit({"event": "reset", "conversationId": cid,
                  "generation": conv.record["generation"]})
        elif cmd == "ask":
            question = str(message.get("question") or "").strip()
            request_id = message.get("id")
            if not question:
                emit({"id": request_id, "conversationId": cid,
                      "event": "error", "message": "empty question"})
                continue
            conv = runner.conversation(cid)
            with runner.state:
                if request_id in runner.cancelled:
                    runner.cancelled.discard(request_id)
                    emit({"id": request_id, "conversationId": cid, "event": "done",
                          "calls": conv.calls, "cost": round(conv.cost, 6),
                          "seconds": 0, "interrupted": True})
                    continue
                conv.cancel.clear()
                runner.active_id, runner.active_cid = request_id, cid
            try:
                try:
                    settings = build_settings(message.get("settings") or {})
                except duet.DuetError as exc:
                    # A rejected setting must not take the backend down with it:
                    # build_settings sits outside runner.ask's own error handling.
                    emit({"id": request_id, "conversationId": cid,
                          "event": "error", "message": str(exc)})
                    continue
                asyncio.run(runner.ask(conv, request_id, question, settings))
            finally:
                with runner.state:
                    runner.active_id = runner.active_cid = None
        else:
            emit({"event": "error", "message": f"unknown cmd {cmd!r}"})


if __name__ == "__main__":
    sys.exit(main())
#!/usr/bin/env python3
"""Per-conversation session records, owned by the backend.

Why the backend owns this and not the webview: `context_for` splices recent
exchanges into every prompt, so history is *executable* prompt state, not display
data. If the client were authoritative, a reloaded tab could feed model context
supplied by client state - including exchanges the user believed a reset had
cleared. The webview keeps ids and titles; what enters a prompt comes from here.

Design decisions worth not re-litigating:

* **Write-through, not write-at-end.** The record is rewritten after every
  canonical mutation. Persisting only at the end of a run leaves a crash window
  between mutating state and reaching disk.
* **Atomic replacement.** Unlike a transcript written once at exit with no
  reader, this file is rewritten every turn and read back by a different process,
  so a torn write is precisely the loss mode. Temp file plus `os.replace`.
* **A per-conversation lock, not just atomic replace.** Reset runs in the reader
  thread and completion in the main loop. `os.replace` gives an untorn file and
  still allows a lost update: reset lands, then an in-flight completion flushes
  stale bytes over it. The lock is held across read-modify-write.
* **Reset is a boundary, not a delete.** It advances `generation`; older
  exchanges stay on disk for the transcript and are excluded from prompts. The
  stronger claim - "pre-reset exchanges exist and are absent from the prompt" -
  is also the one that does not throw away paid-for output.
* **Interrupted exchanges are kept and flagged.** Dropping them would make the
  visible record diverge from what the user saw before pressing Stop. They are
  excluded from prompt context: half a discussion someone cancelled should not
  silently become permanent context for that tab.
* **A version int and try/except, not a schema validator.** Only this program
  writes these files; an unreadable one opens the tab display-only.
"""

from __future__ import annotations

import json
import os
import tempfile
import threading
import time
from pathlib import Path
from typing import Any

VERSION = 1
CONTEXT_EXCHANGES = 3  # how many recent exchanges reach a prompt


def sessions_dir() -> Path:
    """Beside the runs directory, so it inherits "not in a synced folder"."""
    import duet

    return duet.default_runs_dir().parent / "sessions"


def _now() -> str:
    return time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())


def blank(cid: str) -> dict[str, Any]:
    return {
        "version": VERSION,
        "cid": cid,
        "created": _now(),
        "updated": _now(),
        "generation": 0,
        "title": "",
        "exchanges": [],
    }


class SessionStore:
    """One JSON record per conversation id."""

    def __init__(self, directory: Path | None = None) -> None:
        self.dir = Path(directory) if directory else sessions_dir()
        self._locks: dict[str, threading.Lock] = {}
        self._locks_guard = threading.Lock()

    # ---- plumbing ----

    def lock_for(self, cid: str) -> threading.Lock:
        with self._locks_guard:
            lock = self._locks.get(cid)
            if lock is None:
                lock = threading.Lock()
                self._locks[cid] = lock
            return lock

    def path_for(self, cid: str) -> Path:
        safe = "".join(ch for ch in cid if ch.isalnum() or ch in "-_")[:80]
        return self.dir / f"{safe or 'unnamed'}.json"

    def _read(self, cid: str) -> dict[str, Any] | None:
        path = self.path_for(cid)
        if not path.is_file():
            return None
        try:
            record = json.loads(path.read_text(encoding="utf-8"))
        except (json.JSONDecodeError, OSError, UnicodeDecodeError):
            return None
        if not isinstance(record, dict) or record.get("version") != VERSION:
            return None
        if not isinstance(record.get("exchanges"), list):
            return None
        record.setdefault("generation", 0)
        record.setdefault("title", "")
        record.setdefault("cid", cid)
        return record

    def _write(self, record: dict[str, Any]) -> None:
        self.dir.mkdir(parents=True, exist_ok=True)
        record["updated"] = _now()
        path = self.path_for(record["cid"])
        fd, tmp = tempfile.mkstemp(dir=str(self.dir), prefix=".tmp-", suffix=".json")
        try:
            with os.fdopen(fd, "w", encoding="utf-8") as fh:
                json.dump(record, fh, ensure_ascii=False, indent=1)
            os.replace(tmp, path)   # atomic: a reader sees old or new, never half
        except BaseException:
            Path(tmp).unlink(missing_ok=True)
            raise

    # ---- public, each holding the per-conversation lock ----

    def load(self, cid: str) -> dict[str, Any]:
        """The record, or a fresh blank one. Never raises on a bad file."""
        with self.lock_for(cid):
            return self._read(cid) or blank(cid)

    def exists(self, cid: str) -> bool:
        return self.path_for(cid).is_file()

    def append(self, cid: str, question: str, replies: list[tuple[str, str]],
               *, interrupted: bool) -> dict[str, Any]:
        """Record one completed exchange. Called on the main loop."""
        with self.lock_for(cid):
            record = self._read(cid) or blank(cid)
            if not record["title"] and question:
                record["title"] = question[:120]
            record["exchanges"].append({
                "generation": int(record.get("generation", 0)),
                "question": question,
                "replies": [[who, text] for who, text in replies],
                "interrupted": bool(interrupted),
                "at": _now(),
            })
            self._write(record)
            return record

    def new_generation(self, cid: str) -> dict[str, Any]:
        """Reset: a boundary, not a delete. Called on the reader thread."""
        with self.lock_for(cid):
            record = self._read(cid) or blank(cid)
            record["generation"] = int(record.get("generation", 0)) + 1
            self._write(record)
            return record

    def forget(self, cid: str) -> None:
        """Genuinely delete - closing a tab for good, not resetting it."""
        with self.lock_for(cid):
            self.path_for(cid).unlink(missing_ok=True)

    def list(self) -> list[dict[str, Any]]:
        if not self.dir.is_dir():
            return []
        out = []
        for path in sorted(self.dir.glob("*.json")):
            record = self._read(path.stem)
            if record is None:
                continue
            live = [e for e in record["exchanges"]
                    if e.get("generation") == record.get("generation", 0)]
            out.append({
                "cid": record["cid"],
                "title": record.get("title") or "(untitled)",
                "updated": record.get("updated", ""),
                "generation": record.get("generation", 0),
                "exchanges": len(record["exchanges"]),
                "in_context": len([e for e in live if not e.get("interrupted")]),
            })
        return sorted(out, key=lambda r: r["updated"], reverse=True)


# ----------------------------------------------------------------------
# what actually reaches a prompt
# ----------------------------------------------------------------------


def prompt_exchanges(record: dict[str, Any], limit: int = CONTEXT_EXCHANGES) -> list[dict[str, Any]]:
    """The exchanges a prompt may see: current generation, not interrupted.

    Two exclusions, both deliberate. Pre-reset exchanges stay on disk for the
    transcript but a reset means the user wanted a clean slate. An interrupted
    exchange is half a discussion someone hit Stop on, and letting it become
    permanent context for the tab is not what Stop means.
    """
    generation = int(record.get("generation", 0))
    live = [
        e for e in record.get("exchanges", [])
        if int(e.get("generation", 0)) == generation and not e.get("interrupted")
    ]
    return live[-limit:] if limit else live


def display_exchanges(record: dict[str, Any]) -> list[dict[str, Any]]:
    """Everything, for rendering a reloaded tab - including what a reset retired."""
    return list(record.get("exchanges", []))


if __name__ == "__main__":  # a quick look at what is stored
    store = SessionStore()
    rows = store.list()
    print(f"{store.dir}\n")
    if not rows:
        print("no sessions yet")
    for row in rows:
        print(f"  {row['cid'][:12]:14} gen {row['generation']:<3} "
              f"{row['exchanges']:>3} exchange(s), {row['in_context']} in context   "
              f"{row['updated']}  {row['title'][:40]}")
#!/usr/bin/env python3
"""Per-conversation session records, owned by the backend.

Why the backend owns this and not the webview: `context_for` splices recent
exchanges into every prompt, so history is *executable* prompt state, not display
data. If the client were authoritative, a reloaded tab could feed model context
supplied by client state - including exchanges the user believed a reset had
cleared. The webview keeps ids and titles; what enters a prompt comes from here.

Design decisions worth not re-litigating:

* **Write-through, not write-at-end.** The record is rewritten after every
  canonical mutation. Persisting only at the end of a run leaves a crash window
  between mutating state and reaching disk.
* **Atomic replacement.** Unlike a transcript written once at exit with no
  reader, this file is rewritten every turn and read back by a different process,
  so a torn write is precisely the loss mode. Temp file plus `os.replace`.
* **A per-conversation lock, not just atomic replace.** Reset runs in the reader
  thread and completion in the main loop. `os.replace` gives an untorn file and
  still allows a lost update: reset lands, then an in-flight completion flushes
  stale bytes over it. The lock is held across read-modify-write.
* **Reset is a boundary, not a delete.** It advances `generation`; older
  exchanges stay on disk for the transcript and are excluded from prompts. The
  stronger claim - "pre-reset exchanges exist and are absent from the prompt" -
  is also the one that does not throw away paid-for output.
* **Interrupted exchanges are kept and flagged.** Dropping them would make the
  visible record diverge from what the user saw before pressing Stop. They are
  excluded from prompt context: half a discussion someone cancelled should not
  silently become permanent context for that tab.
* **A version int and try/except, not a schema validator.** Only this program
  writes these files; an unreadable one opens the tab display-only.
"""

from __future__ import annotations

import json
import os
import tempfile
import threading
import time
from pathlib import Path
from typing import Any

VERSION = 1
CONTEXT_EXCHANGES = 3  # how many recent exchanges reach a prompt


def sessions_dir() -> Path:
    """Beside the runs directory, so it inherits "not in a synced folder"."""
    import duet

    return duet.default_runs_dir().parent / "sessions"


def _now() -> str:
    return time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())


def blank(cid: str) -> dict[str, Any]:
    return {
        "version": VERSION,
        "cid": cid,
        "created": _now(),
        "updated": _now(),
        "generation": 0,
        "title": "",
        "exchanges": [],
    }


class SessionStore:
    """One JSON record per conversation id."""

    def __init__(self, directory: Path | None = None) -> None:
        self.dir = Path(directory) if directory else sessions_dir()
        self._locks: dict[str, threading.Lock] = {}
        self._locks_guard = threading.Lock()

    # ---- plumbing ----

    def lock_for(self, cid: str) -> threading.Lock:
        with self._locks_guard:
            lock = self._locks.get(cid)
            if lock is None:
                lock = threading.Lock()
                self._locks[cid] = lock
            return lock

    def path_for(self, cid: str) -> Path:
        safe = "".join(ch for ch in cid if ch.isalnum() or ch in "-_")[:80]
        return self.dir / f"{safe or 'unnamed'}.json"

    def _read(self, cid: str) -> dict[str, Any] | None:
        path = self.path_for(cid)
        if not path.is_file():
            return None
        try:
            record = json.loads(path.read_text(encoding="utf-8"))
        except (json.JSONDecodeError, OSError, UnicodeDecodeError):
            return None
        if not isinstance(record, dict) or record.get("version") != VERSION:
            return None
        if not isinstance(record.get("exchanges"), list):
            return None
        record.setdefault("generation", 0)
        record.setdefault("title", "")
        record.setdefault("cid", cid)
        return record

    def _write(self, record: dict[str, Any]) -> None:
        self.dir.mkdir(parents=True, exist_ok=True)
        record["updated"] = _now()
        path = self.path_for(record["cid"])
        fd, tmp = tempfile.mkstemp(dir=str(self.dir), prefix=".tmp-", suffix=".json")
        try:
            with os.fdopen(fd, "w", encoding="utf-8") as fh:
                json.dump(record, fh, ensure_ascii=False, indent=1)
            os.replace(tmp, path)   # atomic: a reader sees old or new, never half
        except BaseException:
            Path(tmp).unlink(missing_ok=True)
            raise

    # ---- public, each holding the per-conversation lock ----

    def load(self, cid: str) -> dict[str, Any]:
        """The record, or a fresh blank one. Never raises on a bad file."""
        with self.lock_for(cid):
            return self._read(cid) or blank(cid)

    def exists(self, cid: str) -> bool:
        return self.path_for(cid).is_file()

    def append(self, cid: str, question: str, replies: list[tuple[str, str]],
               *, interrupted: bool) -> dict[str, Any]:
        """Record one completed exchange. Called on the main loop."""
        with self.lock_for(cid):
            record = self._read(cid) or blank(cid)
            if not record["title"] and question:
                record["title"] = question[:120]
            record["exchanges"].append({
                "generation": int(record.get("generation", 0)),
                "question": question,
                "replies": [[who, text] for who, text in replies],
                "interrupted": bool(interrupted),
                "at": _now(),
            })
            self._write(record)
            return record

    def new_generation(self, cid: str) -> dict[str, Any]:
        """Reset: a boundary, not a delete. Called on the reader thread."""
        with self.lock_for(cid):
            record = self._read(cid) or blank(cid)
            record["generation"] = int(record.get("generation", 0)) + 1
            self._write(record)
            return record

    def forget(self, cid: str) -> None:
        """Genuinely delete - closing a tab for good, not resetting it."""
        with self.lock_for(cid):
            self.path_for(cid).unlink(missing_ok=True)

    def list(self) -> list[dict[str, Any]]:
        if not self.dir.is_dir():
            return []
        out = []
        for path in sorted(self.dir.glob("*.json")):
            record = self._read(path.stem)
            if record is None:
                continue
            live = [e for e in record["exchanges"]
                    if e.get("generation") == record.get("generation", 0)]
            out.append({
                "cid": record["cid"],
                "title": record.get("title") or "(untitled)",
                "updated": record.get("updated", ""),
                "generation": record.get("generation", 0),
                "exchanges": len(record["exchanges"]),
                "in_context": len([e for e in live if not e.get("interrupted")]),
            })
        return sorted(out, key=lambda r: r["updated"], reverse=True)


# ----------------------------------------------------------------------
# what actually reaches a prompt
# ----------------------------------------------------------------------


def prompt_exchanges(record: dict[str, Any], limit: int = CONTEXT_EXCHANGES) -> list[dict[str, Any]]:
    """The exchanges a prompt may see: current generation, not interrupted.

    Two exclusions, both deliberate. Pre-reset exchanges stay on disk for the
    transcript but a reset means the user wanted a clean slate. An interrupted
    exchange is half a discussion someone hit Stop on, and letting it become
    permanent context for the tab is not what Stop means.
    """
    generation = int(record.get("generation", 0))
    live = [
        e for e in record.get("exchanges", [])
        if int(e.get("generation", 0)) == generation and not e.get("interrupted")
    ]
    return live[-limit:] if limit else live


def display_exchanges(record: dict[str, Any]) -> list[dict[str, Any]]:
    """Everything, for rendering a reloaded tab - including what a reset retired."""
    return list(record.get("exchanges", []))


if __name__ == "__main__":  # a quick look at what is stored
    store = SessionStore()
    rows = store.list()
    print(f"{store.dir}\n")
    if not rows:
        print("no sessions yet")
    for row in rows:
        print(f"  {row['cid'][:12]:14} gen {row['generation']:<3} "
              f"{row['exchanges']:>3} exchange(s), {row['in_context']} in context   "
              f"{row['updated']}  {row['title'][:40]}")
# Setting model, effort, profile and lenses

duet does not pick models. It uses whatever each CLI is already configured with,
unless you override it for one run.

## The short version

```bash
python3 duet.py --claude-model fable --claude-effort max \
                --codex-model gpt-5.6-sol --codex-effort high \
                "your question"
```

Every run prints what it resolved, so you never have to guess:

```
Fable:    model=fable  effort=max
Sol:      model=gpt-5.6-sol  effort=high
```

If a line says `from your claude config` / `from your codex config`, duet passed no
flag and the CLI used its own setting.

## Where the defaults live

These two files are the source of truth. **The VS Code chat-box pickers write to
them**, which is why changing the model in the chat box also changes what duet
does on the next run.

| | File | Keys |
| --- | --- | --- |
| Fable (Claude Code) | `~/.claude/settings.json` | `"model"`, `"effortLevel"` |
| Sol (Codex) | `~/.codex/config.toml` | `model`, `model_reasoning_effort` |

Check them any time:

```bash
grep -E '"(model|effortLevel)"' ~/.claude/settings.json
grep -E '^(model|model_reasoning_effort)' ~/.codex/config.toml
```

### Yours, right now

```
~/.claude/settings.json   "model": "opus[1m]",  "effortLevel": "xhigh"
~/.codex/config.toml      model = "gpt-5.6-sol", model_reasoning_effort = "low"
```

**Sol is on `low` and Fable is on `xhigh`.** That is almost certainly why Sol's
answers have been thinner — 100 words to Fable's 591, 2 reasoning points to 6, and
confidence that barely moves. See [Making it a fair fight](#making-it-a-fair-fight).

## Fable (Claude Code)

**Model** — an alias or a full name:

| Alias | |
| --- | --- |
| `fable` | latest Fable |
| `opus` | latest Opus |
| `sonnet` | latest Sonnet |
| `haiku` | latest Haiku |

Full names work too: `claude-fable-5`, `claude-opus-5`. So do context variants:
`opus[1m]` for the 1M-token window.

**Effort** — `low`, `medium`, `high`, `xhigh`, `max`.

**Three ways to change it:**

1. **This run only** — `--claude-model fable --claude-effort max`
2. **In the VS Code chat box** — `/model` to switch model. This persists to
   `settings.json`, so duet picks it up next run.
3. **Permanently** — edit `~/.claude/settings.json`:

   ```json
   { "model": "opus[1m]", "effortLevel": "xhigh" }
   ```

> Note: `--claude-model opus` and `--claude-model 'opus[1m]'` are different. The
> plain alias drops you to the standard context window. If you have deliberately
> chosen `opus[1m]`, pass no `--claude-model` at all and let duet inherit it.

## Sol (Codex)

**Model** — whatever your account offers, e.g. `gpt-5.6-sol`. duet does not
validate it; Codex will reject an unknown name.

**Effort** — `low`, `medium`, `high`. Codex has no effort *flag*, so duet passes
it as a config override: `-c model_reasoning_effort="high"`.

**Three ways to change it:**

1. **This run only** — `--codex-model gpt-5.6-sol --codex-effort high`
2. **In the VS Code chat box** — use the model/effort picker in the composer. It
   writes to `~/.codex/config.toml`.
3. **Permanently** — edit `~/.codex/config.toml`:

   ```toml
   model = "gpt-5.6-sol"
   model_reasoning_effort = "medium"
   ```

**Named profiles** are handy if you switch often. Create
`~/.codex/deep.config.toml`:

```toml
model_reasoning_effort = "high"
```

Then `--codex-model` still works normally, and you can reach the profile directly
with `codex exec -p deep` outside duet. (duet has no `--codex-profile` flag yet —
say the word if you want one.)

## Making it a fair fight

This is the one thing worth getting right. If the two speakers run at different
effort, the comparison stops being about the models and starts being about your
settings — and the better-resourced one looks smarter for no good reason.

Your current split (`xhigh` vs `low`) is a big gap. Level it:

```bash
python3 duet.py --claude-effort high --codex-effort high "your question"
```

Then look at `comparison.md`. If Sol still produces far fewer reasoning points and
assumptions, that is a real difference between the models. Until effort is
matched, you cannot tell the two explanations apart.

## Checking what actually ran

Three places, cheapest first:

```bash
python3 duet.py --dry-run "q"        # exact command lines, invokes nothing
```

The header of every run:

```
Fable:    model=opus[1m]  effort=xhigh
Sol:      model=from your codex config  effort=high
```

And the transcript, which records the model each turn actually used:

```
## Independent answer - Fable
_51s, $0.1147, claude-opus-5_
```

If the transcript names a different model than you expected, the flag did not
apply — check for a typo in the alias.

## Recipes

**Fair comparison** — matched effort, no synthesis, read both sides raw:

```bash
python3 duet.py --claude-effort high --codex-effort high --synthesizer none "..."
```

**Cheap sanity check** — low effort, no exchange, 2 calls:

```bash
python3 duet.py --claude-effort low --codex-effort low --rounds 0 --synthesizer none "..."
```

**Deep on a hard decision** — max effort, two exchanges:

```bash
python3 duet.py --claude-effort max --codex-effort high --rounds 2 "..."
```

**Same model, different personas** — isolates the persona effect from the model
effect, though you lose the cross-model diversity that is the point of duet:

```bash
python3 duet.py --claude-model opus --codex-model gpt-5.6-sol --claude-effort high --codex-effort high "..."
```

## Cost

Effort is the biggest lever on allowance use. Both plans are subscription-based,
so nothing is billed per call — but `max` on a 5-call run consumes far more of your
rate limit than `low`. duet prints a `reported: $x.xxxx` figure from Claude Code;
that is a **notional** equivalent, not a charge.

Use `--fake` or `--dry-run` freely while you are still deciding what to ask. Both
make zero calls.


## Profiles: asking about things that are not code

duet started as an architecture committee, and its default prompts show it. The
`software` profile tells both speakers to cite `file.py:120`, to argue "from the
code", and hands them your repository. Ask it how to build a drone and they will
dutifully grep your repo first — which is exactly what happened here:

> Fable: *"I checked the repo first — `grep -i eiffel|effel|tower` returns nothing"*
> Sol: *"I'll inspect the repository for any project-specific meaning"*

That is the harness narrowing the question, not the models.

```bash
python3 duet.py --profile general "How do I build a drone?"
```

| | software (default) | general |
| --- | --- | --- |
| Repository handed over | yes | no |
| How to ground a claim | cite `file.py:120` | name a measurement, standard, datasheet, regulation… |
| Argue from | "the code" | "what you actually know" |
| Fable's lens | boundaries, load, dependencies, on-call | boundaries, failure under stress, safety and regulatory, cost to run |
| Sol's lens | existing code, seams, migration, shipping | parts and materials, steps in order, difficult joins, what test proves it |

The two **lenses survive the switch**, because they were never really about
software: one asks what breaks and who deals with it, the other asks what it takes
to actually build the thing. For a drone that is a genuinely useful split.

In the chat: `/profile general`. In the panel: the `software`/`general` dropdown.

### Per-profile persona files

`personas/fable.md` and `personas/sol.md` apply to the **software** profile only —
they are written in code vocabulary. For the general profile, create
`personas/general-fable.md` and `personas/general-sol.md`; without them the
built-in domain-neutral lenses are used.

## Lenses off: the purest comparison

```bash
python3 duet.py --profile general --no-personas "How do I build a drone?"
```

Both speakers get **byte-identical prompts** — no lens at all. Any difference in
the answers is then the two models, not something you told them to be. Worth
running once against the same question with lenses on: it shows how much of the
difference you have been attributing to "Claude vs Codex" is actually your own
prompt.

In the chat: `/persona off`. In the panel: `lenses off`.
personas
# Fable (Claude Code)

Edit this file to change how Fable approaches a question. The first heading is
stripped; everything else is injected into every prompt Fable receives.

Keep it about *instinct and emphasis*, not authority. Fable is not the reviewer,
the judge, or the senior one - both advisors are peers, and a human decides.

---

Your instinct is to look at the system as a whole: boundaries, failure modes,
what happens under load or when a dependency dies, security and data handling,
and what this will cost to operate and maintain a year from now. You care about
what breaks and who gets paged at 3am.

Play to that instinct, but answer the actual question. Do not refuse to comment on
implementation detail if it matters to the answer.

When you are uncertain, say so in the same sentence as the claim rather than in a
caveat at the end.

# Sol (Codex)

Edit this file to change how Sol approaches a question. The first heading is
stripped; everything else is injected into every prompt Sol receives.

Keep it about *instinct and emphasis*, not authority. Sol is not the implementer
taking orders, nor the one who defers - both advisors are peers, and a human
decides.

---

Your instinct is to look at what it takes to actually build this: how it fits the
code that already exists, where the seams are, the concrete steps in order, what
the migration looks like, and what test or prototype would prove it works. You
care about whether it can be shipped.

Play to that instinct, but answer the actual question. Do not refuse to comment on
operational or design concerns if they matter to the answer.

When you are uncertain, say so in the same sentence as the claim rather than in a
caveat at the end.
