# Claude Code: Spoken (TTS) Replies via Piper

Makes Claude Code speak its replies out loud using [Piper](https://github.com/rhasspy/piper) (offline neural TTS). Off by default every session; toggle with `/tts` or by asking in chat.

Tested on Fedora Linux, Python 3.11. Requires: `python3`, `pip3`, `jq`, `perl`, `sed`, and `pw-play` (ships with PipeWire, part of `pipewire-utils`/`pipewire`).

**Use `pw-play`, not `aplay`, for playback.** `aplay` talks directly to a raw ALSA hardware device (typically the lowest-numbered card, e.g. a USB headset), which may not be the system's actual active output — on a PipeWire-based desktop (Fedora, most modern distros) the real default sink is managed by PipeWire/WirePlumber and can point somewhere else entirely (e.g. laptop speakers). `aplay` will report success and exit 0 while silently playing into a disconnected/wrong device. `pw-play` routes through PipeWire's actual default sink, matching whatever output the desktop volume control shows as active. Check `wpctl status` to see the current default sink if debugging silent playback.

## 1. Install Piper

```bash
pip3 install --user piper-tts
```

**Immediately resolve and record the exact interpreter this installed into:**

```bash
python3 -c 'import sys; print(sys.executable)'
```

You'll hardcode this absolute path into the hook script in step 3, instead of calling `python3` by name. This matters if you use asdf, pyenv, a venv, or anything else that makes `python3` resolve differently depending on `PATH`/cwd — a hook process can run with a different (often minimal) `PATH` than your interactive shell, so a bare `python3` can silently resolve to the system Python instead, which won't have Piper installed. `pip3 install --user` still installs into that specific interpreter's user site-packages, so only the exact same interpreter binary will find it. This is exactly the failure mode this setup hit on a second machine/session — the script ran, exited 0, and made no sound.

## 2. Download a voice model

Use Piper's own downloader — it resolves the right files from `https://huggingface.co/rhasspy/piper-voices` for you, so there's no need to hand-build URLs:

```bash
mkdir -p ~/.local/share/piper-voices
cd ~/.local/share/piper-voices
python3 -m piper.download_voices en_US-hfc_female-medium
```

This setup currently uses `en_US-hfc_female-medium` (~60 MB onnx + ~5 KB json). Not every voice ships every quality tier — `hfc_female`, for instance, only exists at `medium`, there is no `high` variant. Run `python3 -m piper.download_voices` with no argument to list every available `name-quality` combination before picking one.

Quick sanity test:

```bash
echo "Testing piper voice synthesis." | python3 -m piper -m ~/.local/share/piper-voices/en_US-hfc_female-medium.onnx --output_file /tmp/test.wav
pw-play /tmp/test.wav
```

## 3. The Stop hook script

Save as `~/.claude/scripts/tts-speak.sh` and `chmod +x` it. Claude Code runs this every time it finishes replying (the `Stop` event). It:
- exits immediately if `~/.claude/tts-enabled` doesn't exist (the on/off flag)
- reads `last_assistant_message` directly from the hook's stdin JSON payload — a snapshot of the just-finished reply, already provided by Claude Code, no need to go looking for it
- strips markdown (fenced code blocks, inline code, bold/italic, headers, links) — reading raw code aloud is bad UX
- pipes the cleaned text through Piper, then plays the WAV through `pw-play` (see the `pw-play` vs `aplay` note above)

```bash
#!/usr/bin/env bash
# Stop hook: speaks Claude's last reply via Piper TTS, gated by ~/.claude/tts-enabled.
set -euo pipefail

FLAG="$HOME/.claude/tts-enabled"
VOICE="$HOME/.local/share/piper-voices/en_US-hfc_female-medium.onnx"
# ponytail: hardcoded, not `python3` off PATH — see step 1's note on why a bare
# `python3` can silently resolve to an interpreter without piper installed.
# Replace with the output of: python3 -c 'import sys; print(sys.executable)'
PYTHON="/home/you/.asdf/installs/python/3.11.15/bin/python3.11"

[ -f "$FLAG" ] || exit 0

# last_assistant_message is a snapshot from Stop time. Deriving it from the transcript file
# instead raced: this hook runs async, and by the time it read the (still-growing) transcript
# the next turn had often already appended entries past the reply, extracting nothing.
text=$(jq -r '.last_assistant_message // empty')
[ -n "$text" ] || exit 0

# ponytail: naive markdown stripping, good enough for speech; a real markdown parser is overkill here.
clean=$(printf '%s' "$text" \
  | perl -0777 -pe 's/```.*?```//gs' \
  | sed -E '
      s/`([^`]*)`/\1/g;
      s/\*\*([^*]*)\*\*/\1/g;
      s/\*([^*]*)\*/\1/g;
      s/^#+[[:space:]]*//g;
      s/\[([^]]*)\]\([^)]*\)/\1/g
    ')

[ -n "$clean" ] || exit 0

printf '%s' "$clean" | "$PYTHON" -m piper -m "$VOICE" --output_file - 2>/dev/null | pw-play - 2>/dev/null
```

**Notes:**
- `jq -r '.last_assistant_message // empty'` reads straight from the hook's stdin payload — no transcript file, no JSONL parsing, no race.
- `pw-play -` reads the WAV from stdin, same as `aplay -`.
- both `piper` and `pw-play` have stderr suppressed (`2>/dev/null`) for quiet normal operation — if setting this up fresh, temporarily drop those redirects while testing so real errors (missing module, wrong device, etc.) are visible instead of silently doing nothing.

### Self-check (optional but recommended)

Save as `~/.claude/scripts/test-tts-speak.sh` and `chmod +x`. Feeds a synthetic hook payload through the same markdown-cleanup logic (no audio, no Piper) and asserts the expected output — catches regressions in the regex pipeline.

```bash
#!/usr/bin/env bash
# ponytail self-check: verifies markdown-stripping logic, no audio involved.
set -euo pipefail

text=$(echo '{"last_assistant_message":"Here is `code` and **bold** text.\n```\nignored block\n```\nEnd."}' | jq -r '.last_assistant_message // empty')

clean=$(printf '%s' "$text" \
  | perl -0777 -pe 's/```.*?```//gs' \
  | sed -E '
      s/`([^`]*)`/\1/g;
      s/\*\*([^*]*)\*\*/\1/g;
      s/\*([^*]*)\*/\1/g;
      s/^#+[[:space:]]*//g;
      s/\[([^]]*)\]\([^)]*\)/\1/g
    ')

expected="Here is code and bold text.

End."

if [ "$clean" != "$expected" ]; then
  echo "FAIL: got:"
  printf '%s\n' "$clean"
  exit 1
fi

echo "PASS"
```

Run it after setup: `~/.claude/scripts/test-tts-speak.sh` → should print `PASS`.

## 4. The `/tts` toggle command

Save as `~/.claude/commands/tts.md`. Lets you type `/tts` (toggles), `/tts on`, or `/tts off`.

```markdown
---
description: Toggle spoken (Piper TTS) replies on/off for this session. Usage: /tts [on|off]
---

!`FLAG="$HOME/.claude/tts-enabled"; arg="$ARGUMENTS"; if [ "$arg" = "off" ]; then rm -f "$FLAG"; echo "off"; elif [ "$arg" = "on" ]; then touch "$FLAG"; echo "on"; elif [ -f "$FLAG" ]; then rm -f "$FLAG"; echo "off"; else touch "$FLAG"; echo "on"; fi`

Tell the user in one short sentence that voice replies are now in the state printed above.
```

You can also just ask Claude in chat ("turn on voice replies") — it can `touch`/`rm` the flag file directly via its Bash tool without going through the slash command.

## 5. Wire the hooks into `~/.claude/settings.json`

Merge this into the existing file (don't overwrite other settings/hooks already there):

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "rm -f ~/.claude/tts-enabled"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/scripts/tts-speak.sh",
            "async": true
          }
        ]
      }
    ]
  }
}
```

- `SessionStart` removes the flag file so every new session starts with voice **off**, regardless of how the last session ended.
- `Stop` fires whenever Claude finishes responding (also on `/clear`, resume, and compaction — harmless, since there's usually no fresh assistant text then). `async: true` so playback doesn't block the UI.

Validate the JSON after editing:

```bash
jq -e '.hooks.Stop[0].hooks[0].command' ~/.claude/settings.json
jq -e '.hooks.SessionStart[0].hooks[0].command' ~/.claude/settings.json
```

If `.claude/settings.json` already existed before this Claude Code session started, hooks take effect immediately. Otherwise open `/hooks` once (or restart) to make Claude Code pick up the new file.

## 6. Verify

```bash
touch ~/.claude/tts-enabled   # turn on manually
echo '{"last_assistant_message":"Testing the hook directly."}' | ~/.claude/scripts/tts-speak.sh
rm -f ~/.claude/tts-enabled   # back to off
```

Or just say `/tts on` inside a real session and let Claude reply normally — it should speak.

## Debugging silent failure (hook runs, exit 0, but no sound)

This happened three times while building this setup, every time with no error visible anywhere:

1. **`aplay` writing to the wrong device.** Fixed by switching to `pw-play` (see step 3's note).
2. **`python3` resolving to an interpreter without Piper installed**, because the hook's `PATH` didn't match the interactive shell's (asdf/pyenv/venv-managed Pythons are especially prone to this). Fixed by hardcoding the interpreter's absolute path (see step 1).
3. **A race between the async hook and the transcript file.** The original script derived the reply text by reading the transcript JSONL file itself and slicing "everything after the last user-typed message." Since `Stop` fires with `async: true`, the hook doesn't run at the instant the reply finishes — and if you sent your next message before the hook got around to reading the file, the transcript had already grown past the reply (new user message, new tool calls), so "everything after the last user message" came up empty. The script ran, exited 0, and correctly found nothing to say. Fixed by reading `last_assistant_message` straight from the hook's own stdin payload (a snapshot taken at `Stop` time) instead of re-deriving it from a file that keeps changing underneath the hook.

If it ever goes silent again:
- temporarily edit `tts-speak.sh` to drop the `2>/dev/null` redirects and re-run it manually with a synthetic payload (see step 6) — errors that are invisible during normal hook operation show up immediately.
- to confirm it's specifically the hook-firing path (vs. Piper/pw-play/the script itself), pipe a real payload through the script directly and confirm you hear it, then compare against what actually happens after a real reply.

## Known limitations (ponytail: deliberate, not bugs)

- Markdown stripping is regex-based, not a real parser — good enough for spoken text, not bulletproof against every markdown edge case.
- No length cap on the text sent to Piper — a very long reply will speak in full.
- `Stop` fires on compaction/resume too, not just "real" replies; usually a no-op since there's no new assistant text in the extracted range, but not explicitly filtered out.
