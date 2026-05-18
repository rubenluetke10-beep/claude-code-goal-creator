# Goal Creator — a Claude Code skill

A [Claude Code](https://code.claude.com) skill that translates vague feature ideas into strict, executable `/goal` prompts with a measurable terminal proof as the stop condition.

The skill never executes the goal itself. It is a **prompt translator**: you describe what you want in one fuzzy sentence, and it returns a production-grade `/goal` prompt you can paste into a fresh autonomous Claude Code session.

## What problem this solves

`/goal` runs in an iterative loop until a stop condition reports success. The quality of the loop is entirely determined by the quality of that stop condition. In practice, two failure modes dominate:

1. **Vibe metrics** — prompts that say things like "the plugin is cleanly implemented and works well". The model can declare itself done without producing anything verifiable.
2. **Fragile or stale proof commands** — hardcoded timestamps that break on a next-day restart, multi-level quoted PowerShell/Bash one-liners that blow up on a single special character.

Goal Creator enforces four rules on every prompt it emits:

1. A binary terminal proof (exit-code-based) as the stop condition.
2. No vibe words.
3. End state, not procedure.
4. Robust proof commands — no hardcoded dates, no fragile nested-quoting one-liners.

## Example

**You say:** "I want a new STT plugin."

**Goal Creator returns:**

```text
/goal New speech-to-text plugin `stt_whisper` is runnable in the plugin directory and returns a non-empty transcript string on a test audio file.

Definition of Done — ALL conditions must hold:

1. File `plugins/stt_whisper/__init__.py` exists and exports a class `STTWhisperPlugin` with a method `transcribe(audio_path: str) -> str`.
2. File `plugins/stt_whisper/plugin.toml` exists with fields `name`, `version`, `entrypoint`.
3. File `tests/test_stt_whisper.py` exists and contains a test case `test_transcribes_sample_wav`.
4. Test asset `tests/fixtures/sample.wav` exists.

Terminal proof (from repo root, all exit 0):

  python -c "from plugins.stt_whisper import STTWhisperPlugin; print('IMPORT_OK')"
  pytest tests/test_stt_whisper.py -v

Expected outputs:
- Command 1 → stdout contains `IMPORT_OK`
- Command 2 → stdout contains `1 passed`

Constraints:
- No new system dependencies outside `requirements.txt`.

Out of scope:
- Performance tuning.
- Multi-language support.
- UI integration.
```

You then paste that into a fresh Claude Code session as `/goal …`.

## Install

Drop the `goal-creator/` folder into your Claude Code skills directory:

- macOS / Linux: `~/.claude/skills/goal-creator/`
- Windows: `%USERPROFILE%\.claude\skills\goal-creator\`

Claude Code auto-discovers skills on session start. No registration step.

## Trigger phrases

The skill activates when you say things like:

- "Goal Creator …"
- "build me a /goal prompt for …"
- "turn this into a /goal"
- "I need a goal prompt that …"
- "wrap this as /goal"

It explicitly does **not** activate on plain "implement X" or "fix Y" — those are direct execution requests, not `/goal` prompt requests.

## How it works internally

On every invocation the skill:

1. Fetches the current `/goal` docs from `code.claude.com/docs/en/goal` so its understanding of the mechanism stays current. (Knowledge feeds into prompt quality, not into the prompt text.)
2. Analyzes the user idea, classifies the task type (new function, bugfix, refactor, plugin, CLI tool, web endpoint, build, etc.).
3. Picks an appropriate proof command from a heuristic table.
4. Generates the prompt against a strict template (Definition of Done, Terminal proof, Expected outputs, Constraints, Out of scope).
5. Runs a self-check list before emitting: no vibe words, no hardcoded dates, no fragile nested-quoting one-liners, no doc links bleeding into the output.

## License

MIT — see [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. The thing this skill needs most is more proof-construction patterns for task types that aren't yet in the heuristic table (mobile, ML training, infrastructure, etc.).
