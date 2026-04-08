---
name: home-assistant-tts
description: Speak a message aloud through all Home Assistant speakers using the HA TTS REST API. Use when the user asks to announce something, read something out loud, or speak on the house speakers.
---

# /home-assistant-tts — Speak via Home Assistant TTS

Announce a message on all Home Assistant media players using the TTS REST API.

## Prerequisites

Read your HA credentials from `/workspace/group/home_assistant.md`:

```bash
# Extract URL and token from the markdown file
HA_URL=$(grep -oP '(?<=\*\*URL\*\*: ).*' /workspace/group/home_assistant.md | tr -d '[:space:]')
HA_TOKEN=$(grep -oP '(?<=\*\*Token\*\*: ).*' /workspace/group/home_assistant.md | tr -d '[:space:]')
```

All `curl` calls must include `--noproxy "*"` (container proxy blocks direct LAN access).

## Steps

### 1. Get all media player entity IDs

```bash
curl -s --noproxy "*" -X GET "${HA_URL}/api/states" \
  -H "Authorization: Bearer ${HA_TOKEN}" \
  -H "Content-Type: application/json" \
  | grep -o '"entity_id":"media_player\.[^"]*"' \
  | grep -o 'media_player\.[^"]*'
```

Collect all returned entity IDs. If none found, report to user and stop.

### 2. Detect available TTS engine

```bash
curl -s --noproxy "*" -X GET "${HA_URL}/api/states" \
  -H "Authorization: Bearer ${HA_TOKEN}" \
  -H "Content-Type: application/json" \
  | grep -o '"entity_id":"tts\.[^"]*"' \
  | grep -o 'tts\.[^"]*'
```

Pick the first result (e.g. `tts.home_assistant_cloud`, `tts.piper`). If none, use the fallback in step 3b.

### 3a. Speak using `tts.speak` (modern HA, 2024+)

```bash
curl -s --noproxy "*" -X POST "${HA_URL}/api/services/tts/speak" \
  -H "Authorization: Bearer ${HA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"entity_id\": \"<TTS_ENTITY_ID>\",
    \"media_player_entity_id\": [<COMMA_SEPARATED_QUOTED_ENTITY_IDS>],
    \"message\": \"<MESSAGE>\"
  }"
```

### 3b. Fallback: `tts.google_translate_say`

```bash
curl -s --noproxy "*" -X POST "${HA_URL}/api/services/tts/google_translate_say" \
  -H "Authorization: Bearer ${HA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"entity_id\": [<COMMA_SEPARATED_QUOTED_ENTITY_IDS>],
    \"message\": \"<MESSAGE>\"
  }"
```

## Error handling

- Non-2xx response or `{"message":"..."}` error body → report status code and message to user.
- Missing credentials → tell user to check `/workspace/group/home_assistant.md`.

## Example invocations

> "Announce that dinner is ready"
> "Tell the house the meeting starts in 5 minutes"
> "Read out the weather on all speakers"
