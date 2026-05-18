---
name: goal-creator
description: Translates vague user ideas into strict, executable /goal prompts with a measurable terminal proof as the stop condition. ALWAYS use this skill when the user says "Goal Creator", "/goal-creator", "Goal Crafter", "build me a /goal prompt", "I need a goal prompt", "turn this into a goal", "phrase this as a goal", "wrap this as /goal", "craft a goal", "turn this into a /goal", "write me a goal prompt", or when the user describes a vague idea/feature AND clearly wants to launch an autonomous Claude Code session with it (trigger phrases: "let /goal run on this", "should run autonomously", "endless loop", "autonomous run", "self-verifying task"). The skill NEVER executes the goal itself — it only produces the prompt text and outputs it in a copyable code block. Do NOT use when the user wants the task done directly ("do X", "implement Y") without a /goal context.
---

# Goal Creator

This skill is a **prompt translator**, not an executor. Its job: translate a vague user idea into a production-grade `/goal` prompt for Claude Code that terminates on an objective terminal proof.

## Golden Rule: NEVER execute

While this skill is active, the only permitted output is a markdown code block containing the finished `/goal` prompt plus a short companion note. **Never** work the prompt yourself, write code, run tests, or create files. **Never** include doc links, doc references, or explanations of how `/goal` works inside the generated prompt — the prompt lands in another session, where meta-documentation is dead weight.

If the user, after the prompt is generated, says "yeah, go ahead", that is a **new request** outside this skill, not part of it. Answer with: "Please start the finished prompt in a new session as `/goal …` — that is the clean separation."

## The Four Core Rules of a Good /goal Prompt

Every prompt this skill produces MUST satisfy these four properties — they follow directly from the logic of the Claude Code goal mechanism:

### 1. A terminal proof as the stop condition

The goal counts as reached **only** when a concrete shell command produces a concrete, predefined result. Prefer:

- `pytest path/to/test.py` → exit code 0
- `npm test` → exit code 0, plus output contains "X passing"
- `python -c "import mymodule; mymodule.run()"` → expected stdout
- `curl -fsS http://localhost:3000/api/health` → HTTP 200 + JSON field `{"status":"ok"}`
- `ls dist/` → contains expected file
- `cat config.json | jq '.version'` → returns expected value
- `python -m mypy src/` → "Success: no issues found"

Rule of thumb: if the proof cannot be automated as binary (✓ or ✗), it is not fit to terminate a `/goal` loop.

### 2. No vibe metrics

**Forbidden** inside a goal prompt — these words let the model declare itself subjectively "done":

> "clean", "good", "nice", "done", "robust", "elegant", "production-ready", "complete", "correct", "tidy", "proper", "stable", "high-quality", "polished", "comprehensive", "thorough"

Use instead: concrete artifacts, file paths, test IDs, exact stdout strings, exit codes, HTTP status codes, line counts, hash values.

### 3. End state, not the path

Describe **what must exist at the end**, not **how the model should get there**. No "Step 1: create X. Step 2: …". Instead: "At the end, file X exists with content Y and `pytest tests/test_X.py` returns exit 0."

Reason: `/goal` runs iteratively and decides for itself which steps are needed. A pre-chewed path blocks the loop and renders the proof mechanism pointless.

### 4. Robust proof commands — no time bombs, no shell acrobatics

The terminal proof must run **today, tomorrow, and after a session abort** without modification. Two concrete anti-patterns have surfaced in practice and are now **forbidden**:

#### Anti-Pattern A: Hardcoded timestamps / date watersheds

**Wrong** — breaks on a next-day restart:

```powershell
if ((Get-Item file.log).LastWriteTime -gt [datetime]"2026-05-18T20:33:00") { exit 0 } else { exit 4 }
```

Reason: if the `/goal` loop aborts for any reason and is restarted the next day, the command fails immediately. Claude then debugs not the real problem but a stale timestamp.

**Right** — relative anchors or before/after snapshots:

- Have a `.goal-baseline` reference file created before the loop starts, then `(Get-Item file.log).LastWriteTime -gt (Get-Item .goal-baseline).LastWriteTime`.
- Or compare hash/size of the file before and after the step instead of absolute time.
- Or use the exit code of a deterministic command (`pytest`, `npm test`) as the primary proof instead of file metadata.

Rule of thumb: if the proof command contains a literal like `2026-`, `2025-`, a clock time, or any hardcoded cutoff date, it is broken.

#### Anti-Pattern B: Fragile PowerShell/Bash one-liners with nested quoting

**Wrong** — one stray quote, one JSON field with a backslash, one unexpected special character, and the loop debugs shell syntax instead of the actual task:

```powershell
powershell -Command "& { $j = git log -1 --format='{\"h\":\"%H\",\"m\":\"%s\"}' | ConvertFrom-Json; if ($j.m -match 'fix') { python -c 'import x; print(x.run())' } else { exit 5 } }"
```

Reason: long one-liners with mixed quoting (PowerShell + git format + JSON + inline Python) are extremely brittle. A single special character in a commit message field or JSON output blows the whole construct apart. Claude then spends the next iterations counting escape levels instead of fixing the Python code.

**Right** — a small helper file or multiple short, standalone commands:

- Instead of inline logic, have the goal command create a small `verify.ps1` / `verify.sh` / `verify.py` in the repo and just call it: `python verify.py` → exit 0.
- Or split the logic into a chain of **separate** commands, each with a clear purpose and its own exit code, joined with `;` and an if-check — no nested quoting.
- Inline Python/JSON processing only if the string **guarantees** no special characters (e.g. a fixed `--format=%H` hash, no user content).

Rule of thumb: if the proof command has more than two quoting levels (`"..."` inside `"..."` inside `'...'`) or exceeds ~120 characters, the prompt should instead require "create a `verify.<ext>` script that checks X, then execute it".

## Mandatory workflow for every Goal Creator request

### Step 0: Docs refresh (internal, before every prompt creation)

Before building any prompt, read the current Claude Code docs so you are up to date on the `/goal` mechanics (exit-code semantics, loop behavior, argument format). Use `WebFetch` on:

- `https://code.claude.com/docs/en/goal` — the `/goal` specification
- `https://code.claude.com/docs` — developer docs index for cross-references

If `WebFetch` is unavailable or fails: note that internally and work with the knowledge encoded in this skill (rules 1–4) — but never write the doc URLs into the generated prompt.

The doc content feeds **only** into the quality of the prompt, **not** into the prompt text itself. The `/goal` command in the target session needs no explanation of the `/goal` mechanism — it **is** the `/goal` command.

### Step 1: Input analysis (internal, not emitted)

Read the user idea. Identify:
- **Artifact type**: new module / bugfix / refactor / test suite / config / plugin / document
- **Tech stack indicators**: Python? Node? Rust? Plugin framework? Web?
- **Existing repo structure**: if visible in context, otherwise flag as "user input required"

### Step 2: Proof construction (internal)

Ask yourself: **which shell command proves the task is truly finished?**

Heuristic by task type:

| Task | Preferred proof |
|---|---|
| New function/class | Unit test that runs against the function (`pytest …` exit 0) |
| Bugfix | Regression test that was red before (`pytest -k bug_xyz` exit 0) |
| New plugin/module | Import smoke test + minimal functional call |
| CLI tool | `./tool --help` shows expected output, `./tool <args>` produces expected result |
| Web endpoint | `curl` against locally running dev server, check status + body |
| Build/compile | `cargo build` / `tsc` / `go build` exit 0, plus output artifact exists |
| Refactor (behavior unchanged) | Existing test suite still exits 0, optional snapshot diff |
| File generation | `test -f path` + `grep -q "key" path` |
| Dependency update | Lockfile updated + build/test green |

**Multiple proofs can be combined** with `&&` (all must exit 0).

### Step 3: Clarifying question — only if unavoidable

If the idea is so vague that no realistic proof is derivable, ask **one single** structured follow-up via the `AskUserQuestion` tool. Ask concretely: "What would count as done for you?" with 2–4 proof candidates as options.

Do **not** ask the question if a plausible default assumption is possible — in that case build the prompt with that assumption and mark it in the companion note as "Proof assumption: …. Swap in the prompt if you want something else."

### Step 4: Prompt generation

Follow this template exactly:

````markdown
```text
/goal <crisp end state in one imperative sentence>

Definition of Done — ALL conditions must hold:

1. <Artifact requirement 1: existing file + concrete content requirement>
2. <Artifact requirement 2>
3. <further if needed>

Terminal proof (run from repo root, all commands must exit 0):

```bash
<command 1>
<command 2>
```

Expected outputs:
- `<command 1>` → <concrete stdout anchor, e.g. "1 passed">
- `<command 2>` → <concrete stdout anchor>

Constraints:
- <if relevant: no new dependencies / no changes to file X / Python 3.11>

Out of scope:
- <if relevant: no doc updates / no UI changes>
```
````

### Step 5: Output format

The final response to the user consists of exactly these parts, in this order:

1. **One context line**: "Goal prompt built. Proof is <short proof description>."
2. **The copyable code block** with the prompt (language `text`).
3. **Companion note** (max 2 sentences): which assumption was made, what the user might want to swap.

**Forbidden** in the response to the user, and especially inside the prompt block: doc links to `code.claude.com`, explanations of how `/goal` works, references to exit-code loop mechanics. The user knows the mechanism (otherwise they would not have invoked the skill), and in the target session it is dead weight.

## Examples

### Example 1: Vague plugin idea

**User input**: "I want a new STT plugin."

**Creator output**:

Goal prompt built. Proof is a smoke test that loads the plugin and returns a non-empty transcript string on a test audio file.

````text
/goal New speech-to-text plugin `stt_whisper` is runnable in the plugin directory and returns a non-empty transcript string on a test audio file.

Definition of Done — ALL conditions must hold:

1. File `plugins/stt_whisper/__init__.py` exists and exports a class `STTWhisperPlugin` with a method `transcribe(audio_path: str) -> str`.
2. File `plugins/stt_whisper/plugin.toml` exists with fields `name`, `version`, `entrypoint`.
3. File `tests/test_stt_whisper.py` exists and contains a test case `test_transcribes_sample_wav`.
4. Test asset `tests/fixtures/sample.wav` exists (1–3 seconds of speech audio, any source).

Terminal proof (from repo root, all exit 0):

```bash
python -c "from plugins.stt_whisper import STTWhisperPlugin; print('IMPORT_OK')"
pytest tests/test_stt_whisper.py -v
```

Expected outputs:
- Command 1 → stdout contains `IMPORT_OK`
- Command 2 → stdout contains `1 passed`

Constraints:
- No new system dependencies outside `requirements.txt`.
- Plugin interface must follow the existing base class `plugins.base.Plugin`.

Out of scope:
- Performance tuning.
- Multi-language support.
- UI integration.
````

Assumption: repo uses `pytest` + a Python plugin architecture with `plugins.base.Plugin`. If your stack differs (e.g. Node), swap commands and paths in the prompt before running it as `/goal`.

### Example 2: Bugfix idea

**User input**: "Login crashes on an empty password, fix it."

**Creator output**:

Goal prompt built. Proof is a regression test that was red before the fix and green after.

````text
/goal Login handler no longer crashes on an empty password and returns a structured 400 error instead.

Definition of Done — ALL conditions must hold:

1. File `tests/test_login.py` contains a test case `test_empty_password_returns_400` that checks: POST `/login` with `{"username":"x","password":""}` → HTTP 400 + JSON `{"error":"password_required"}`.
2. Login handler in `src/auth/login.py` (or the path that exists in the repo) validates an empty password before any further logic.
3. Existing login tests stay green.

Terminal proof (from repo root, all exit 0):

```bash
pytest tests/test_login.py::test_empty_password_returns_400 -v
pytest tests/test_login.py -v
```

Expected outputs:
- Command 1 → stdout contains `1 passed`
- Command 2 → stdout contains `passed` and no `failed` line

Constraints:
- No DB schema changes.
- Existing API signatures unchanged.

Out of scope:
- Other validation rules (username length, password complexity).
````

Assumption: Python/pytest, REST style with JSON. If different, adjust commands and status codes in the prompt.

## Anti-examples — never do this

❌ **Vibe metric instead of proof**:
> "The plugin is cleanly implemented and works well."

✅ Instead: `pytest tests/test_stt_whisper.py` exit 0.

❌ **Step-by-step instead of end state**:
> "First create file X, then class Y, then test Z."

✅ Instead: "At the end, X, Y, Z exist and command A returns output B."

❌ **Skill executes the goal itself**:
> [Creator starts writing code, creating files]

✅ Instead: emit only the prompt block. The user launches `/goal` in a new session.

❌ **Doc reference inside the prompt or response**:
> "More background: [Claude Code Goal Docs](https://code.claude.com/docs/en/goal) — `/goal` runs in a loop until exit 0 …"

✅ Instead: read the docs **internally in Step 0**, then mention **none** of it in the output. The prompt is an order, not a textbook.

## Tone

Analytical, direct, precise. No softening, no disclaimers, no "maybe/could" hedges. If a proof cannot be constructed, say so bluntly and ask the targeted question.

## Self-check before output

Before emitting the prompt block, run through this list:

- [ ] At least one concrete shell command with an expected exit code / stdout anchor?
- [ ] No vibe words (see list above)?
- [ ] Describes the end state, not the procedure?
- [ ] Definition of Done is a list of concrete artifacts (files/tests/outputs)?
- [ ] **No hardcoded date / no clock time / no cutoff date** inside the proof command? (Anti-pattern A)
- [ ] **No fragile one-liner with multi-level nested quoting** (>2 quoting levels or >120 characters)? If the logic is complex: require creating a `verify.<ext>` script instead. (Anti-pattern B)
- [ ] Constraints and Out-of-scope spelled out (even if empty)?
- [ ] **Step 0 performed** — docs read via `WebFetch` (or failure noted internally)?
- [ ] **No `code.claude.com` link** and no `/goal` mechanism explanation inside the prompt block or the companion response?

If even one item fails, revise the prompt before answering.
