# Claude Code: Spoken (TTS) Replies via Piper

Makes Claude Code speak its replies out loud using [Piper](https://github.com/rhasspy/piper) (offline neural TTS). Off by default every session; toggle with `/tts` or by asking in chat.

Tested on Fedora Linux, Python 3.11. Requires: `python3`, `pip3`, `jq`, `perl`, `sed`, and an audio player (`aplay`, from `alsa-utils`).

## 1. Install Piper

```bash
pip3 install --user piper-tts
```

## 2. Download a voice model

Voices live at `https://huggingface.co/rhasspy/piper-voices/tree/main`. This setup uses `en_US-lessac-high` (~109 MB onnx + ~5 KB json). Swap the path segment (`en/en_US/lessac/high`) for any other voice/quality tier.

```bash
mkdir -p ~/.local/share/piper-voices
curl -sL -o ~/.local/share/piper-voices/en_US-lessac-high.onnx \
  "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/high/en_US-lessac-high.onnx"
curl -sL -o ~/.local/share/piper-voices/en_US-lessac-high.onnx.json \
  "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_US/lessac/high/en_US-lessac-high.onnx.json"
```

Quick sanity test:

```bash
echo "Testing piper voice synthesis." | python3 -m piper -m ~/.local/share/piper-voices/en_US-lessac-high.onnx --output_file /tmp/test.wav
aplay /tmp/test.wav
```

## 3. The Stop hook script

Save as `~/.claude/scripts/tts-speak.sh` and `chmod +x` it. Claude Code runs this every time it finishes replying (the `Stop` event). It:
- exits immediately if `~/.claude/tts-enabled` doesn't exist (the on/off flag)
- reads `transcript_path` from the hook's stdin JSON payload
- pulls out all `text` blocks from the assistant's messages since the last user message (a single reply can span several JSONL lines when tool calls are interleaved)
- strips markdown (fenced code blocks, inline code, bold/italic, headers, links) — reading raw code aloud is bad UX
- pipes the cleaned text through Piper, then plays the WAV through `aplay`

```bash
#!/usr/bin/env bash
# Stop hook: speaks Claude's last reply via Piper TTS, gated by ~/.claude/tts-enabled.
set -euo pipefail

FLAG="$HOME/.claude/tts-enabled"
VOICE="$HOME/.local/share/piper-voices/en_US-lessac-high.onnx"

[ -f "$FLAG" ] || exit 0

transcript=$(jq -r '.transcript_path // empty')
[ -n "$transcript" ] && [ -f "$transcript" ] || exit 0

text=$(jq -rs '
  [.[] | select(.type=="user" or .type=="assistant")] as $msgs
  | ($msgs | map(.type) | rindex("user")) as $lastUser
  | $msgs[($lastUser // -1)+1:]
  | map(select(.type=="assistant") | .message.content[]? | select(.type=="text") | .text)
  | join("\n")
' "$transcript")

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

printf '%s' "$clean" | python3 -m piper -m "$VOICE" --output_file - 2>/dev/null | aplay -q - 2>/dev/null
```

**Note:** the `jq -rs` (raw output, `-r`) flag is required — without it, jq returns a JSON-encoded string (literal quotes and `\n` escapes) instead of plain text.

### Self-check (optional but recommended)

Save as `~/.claude/scripts/test-tts-speak.sh` and `chmod +x`. Feeds a synthetic transcript through the same extraction/cleanup logic (no audio, no Piper) and asserts the expected output — catches regressions like the missing `-r` flag above.

```bash
#!/usr/bin/env bash
# ponytail self-check: verifies transcript text-extraction + markdown stripping, no audio involved.
set -euo pipefail

tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT

cat > "$tmp" <<'EOF'
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"hi"}]}}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"thinking","thinking":"..."}]}}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"tool_use","name":"Bash"}]}}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"Here is `code` and **bold** text.\n```\nignored block\n```\nEnd."}]}}
EOF

text=$(jq -rs '
  [.[] | select(.type=="user" or .type=="assistant")] as $msgs
  | ($msgs | map(.type) | rindex("user")) as $lastUser
  | $msgs[($lastUser // -1)+1:]
  | map(select(.type=="assistant") | .message.content[]? | select(.type=="text") | .text)
  | join("\n")
' "$tmp")

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
echo '{"transcript_path":"/path/to/some/transcript.jsonl"}' | ~/.claude/scripts/tts-speak.sh
rm -f ~/.claude/tts-enabled   # back to off
```

Or just say `/tts on` inside a real session and let Claude reply normally — it should speak.

## Known limitations (ponytail: deliberate, not bugs)

- Markdown stripping is regex-based, not a real parser — good enough for spoken text, not bulletproof against every markdown edge case.
- No length cap on the text sent to Piper — a very long reply will speak in full.
- `Stop` fires on compaction/resume too, not just "real" replies; usually a no-op since there's no new assistant text in the extracted range, but not explicitly filtered out.
