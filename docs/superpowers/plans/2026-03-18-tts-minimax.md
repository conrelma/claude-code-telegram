# TTS via MiniMax Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add text-to-speech so Claude's responses can be sent back as Telegram voice messages, using the MiniMax TTS API.

**Architecture:** A new `TTSHandler` class in `voice_handler.py` calls the MiniMax HTTP API with response text, receives hex-encoded MP3 bytes, and the orchestrator sends them via `reply_voice()`. TTS triggers automatically on voice-input responses, or when the user toggles `/voice` on. Text is always sent first; voice follows.

**Tech Stack:** MiniMax TTS API (HTTP + urllib stdlib), python-telegram-bot `reply_voice()`, Pydantic Settings for config.

---

### Task 1: Add TTS config fields to Settings

**Files:**
- Modify: `src/config/settings.py:169-197` (voice settings block)
- Modify: `src/config/settings.py:391-400` (voice_provider validator)
- Modify: `src/config/settings.py:490-525` (voice properties)

- [ ] **Step 1: Add TTS fields after the existing voice settings (after line 197)**

Add these fields to the `Settings` class, after `voice_max_file_size_mb`:

```python
    # TTS (text-to-speech) settings
    tts_enabled: bool = Field(
        False, description="Enable text-to-speech for responses"
    )
    tts_provider: Literal["minimax"] = Field(
        "minimax", description="TTS provider"
    )
    tts_model: str = Field(
        "speech-2.8-hd", description="MiniMax TTS model"
    )
    tts_voice_id: str = Field(
        "English_Graceful_Lady",
        description="MiniMax voice ID for TTS output",
    )
    minimax_api_key: Optional[SecretStr] = Field(
        None, description="MiniMax API key for TTS"
    )
```

- [ ] **Step 2: Add minimax_api_key_str property (after openai_api_key_str, ~line 497)**

```python
    @property
    def minimax_api_key_str(self) -> Optional[str]:
        """Get MiniMax API key as string."""
        return (
            self.minimax_api_key.get_secret_value() if self.minimax_api_key else None
        )
```

- [ ] **Step 3: Verify settings load**

Run: `cd ~/cowork/claude-code-telegram && .venv/bin/python -c "from src.config.settings import Settings; print('OK')"`
Expected: `OK`

- [ ] **Step 4: Commit**

```bash
cd ~/cowork/claude-code-telegram
git add src/config/settings.py
git commit -m "feat: add TTS config fields (MiniMax provider)" --no-gpg-sign
```

---

### Task 2: Create TTSHandler class

**Files:**
- Modify: `src/bot/features/voice_handler.py` (add TTSHandler class after VoiceHandler)

- [ ] **Step 1: Add TTSHandler class at the end of voice_handler.py (after line 189)**

```python

class TTSHandler:
    """Convert text to speech using MiniMax TTS API."""

    TTS_ENDPOINT = "https://api.minimaxi.com/v1/t2a_v2"
    MAX_CHARS = 10_000

    def __init__(self, config: Settings):
        self.config = config

    async def synthesize(self, text: str) -> Optional[bytes]:
        """Convert text to MP3 bytes. Returns None on failure."""
        api_key = self.config.minimax_api_key_str
        if not api_key:
            logger.warning("TTS skipped: no MiniMax API key")
            return None

        # Strip HTML tags and trim to limit
        import re

        clean = re.sub(r"<[^>]+>", "", text).strip()
        if not clean:
            return None
        if len(clean) > self.MAX_CHARS:
            clean = clean[: self.MAX_CHARS]

        payload = json.dumps(
            {
                "model": self.config.tts_model,
                "text": clean,
                "stream": False,
                "voice_setting": {
                    "voice_id": self.config.tts_voice_id,
                    "speed": 1.0,
                    "vol": 1.0,
                    "pitch": 0,
                },
                "audio_setting": {
                    "sample_rate": 32000,
                    "bitrate": 128000,
                    "format": "mp3",
                    "channel": 1,
                },
                "output_format": "hex",
            }
        ).encode()

        req = urllib.request.Request(
            self.TTS_ENDPOINT,
            data=payload,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )

        try:
            loop = asyncio.get_running_loop()
            resp_bytes = await loop.run_in_executor(
                None, lambda: urllib.request.urlopen(req, timeout=30).read()
            )
            resp = json.loads(resp_bytes)
        except Exception as exc:
            logger.warning("TTS API request failed", error=str(exc))
            return None

        status_code = resp.get("base_resp", {}).get("status_code", -1)
        if status_code != 0:
            logger.warning(
                "TTS API error",
                status_code=status_code,
                status_msg=resp.get("base_resp", {}).get("status_msg"),
            )
            return None

        hex_audio = resp.get("data", {}).get("audio", "")
        if not hex_audio:
            logger.warning("TTS returned empty audio")
            return None

        return bytes.fromhex(hex_audio)
```

- [ ] **Step 2: Add required imports at the top of voice_handler.py**

Add to the existing imports at the top of the file:

```python
import asyncio
import json
import urllib.request
```

The file already imports `from typing import Any, Optional` and `structlog`, and `from src.config.settings import Settings`.

- [ ] **Step 3: Verify import works**

Run: `cd ~/cowork/claude-code-telegram && .venv/bin/python -c "from src.bot.features.voice_handler import TTSHandler; print('OK')"`
Expected: `OK`

- [ ] **Step 4: Commit**

```bash
cd ~/cowork/claude-code-telegram
git add src/bot/features/voice_handler.py
git commit -m "feat: add TTSHandler class for MiniMax TTS" --no-gpg-sign
```

---

### Task 3: Register TTSHandler in FeatureRegistry

**Files:**
- Modify: `src/bot/features/registry.py:19` (import)
- Modify: `src/bot/features/registry.py:81-90` (init block)
- Modify: `src/bot/features/registry.py:133-135` (getter)
- Modify: `src/bot/features/__init__.py` (export)

- [ ] **Step 1: Add TTSHandler import in registry.py (line 19)**

Change:
```python
from .voice_handler import VoiceHandler
```
To:
```python
from .voice_handler import TTSHandler, VoiceHandler
```

- [ ] **Step 2: Add TTS initialization after the voice handler block (~after line 90)**

```python
        # TTS (text-to-speech) - requires MiniMax API key
        if self.config.tts_enabled and self.config.minimax_api_key:
            try:
                self.features["tts_handler"] = TTSHandler(config=self.config)
                logger.info("TTS handler feature enabled")
            except Exception as e:
                logger.error("Failed to initialize TTS handler", error=str(e))
```

- [ ] **Step 3: Add getter method (after get_voice_handler, ~line 135)**

```python
    def get_tts_handler(self) -> Optional["TTSHandler"]:
        """Get TTS handler feature."""
        return self.get_feature("tts_handler")
```

- [ ] **Step 4: Update `__init__.py` exports**

Change:
```python
from .voice_handler import ProcessedVoice, VoiceHandler
```
To:
```python
from .voice_handler import ProcessedVoice, TTSHandler, VoiceHandler
```

Add `"TTSHandler"` to the `__all__` list.

- [ ] **Step 5: Commit**

```bash
cd ~/cowork/claude-code-telegram
git add src/bot/features/registry.py src/bot/features/__init__.py
git commit -m "feat: register TTSHandler in feature registry" --no-gpg-sign
```

---

### Task 4: Add /voice command and wire TTS into orchestrator

**Files:**
- Modify: `src/bot/orchestrator.py:304-313` (register handler)
- Modify: `src/bot/orchestrator.py:410-422` (bot commands)
- Modify: `src/bot/orchestrator.py` (add command handler method)
- Modify: `src/bot/orchestrator.py:1301-1340` (agentic_voice — send TTS after response)
- Modify: `src/bot/orchestrator.py:1022-1061` (agentic_text — send TTS if toggled)

- [ ] **Step 1: Add /voice command registration (in `_register_agentic_handlers`, after the "restart" entry ~line 310)**

Add to the handlers list:
```python
            ("voice", self.agentic_voice_toggle),
```

- [ ] **Step 2: Add /voice to bot commands (in `get_bot_commands`, after the "restart" BotCommand ~line 419)**

```python
                BotCommand("voice", "Toggle voice responses on/off"),
```

- [ ] **Step 3: Add the `/voice` toggle handler method (after `agentic_verbose`, ~line 579)**

```python
    async def agentic_voice_toggle(
        self, update: Update, context: ContextTypes.DEFAULT_TYPE
    ) -> None:
        """Toggle always-on TTS: /voice."""
        current = context.user_data.get("tts_always_on", False)
        context.user_data["tts_always_on"] = not current
        state = "ON" if not current else "OFF"
        await update.message.reply_text(f"Voice responses: <b>{state}</b>", parse_mode="HTML")
```

- [ ] **Step 4: Add a helper method to send TTS (after `agentic_voice_toggle`)**

```python
    async def _maybe_send_tts(
        self,
        update: Update,
        context: ContextTypes.DEFAULT_TYPE,
        formatted_messages: list,
    ) -> None:
        """Send a voice message if TTS is active."""
        features = context.bot_data.get("features")
        tts_handler = features.get_tts_handler() if features else None
        if not tts_handler:
            return

        # Combine all message texts
        full_text = "\n".join(
            m.text for m in formatted_messages if m.text and m.text.strip()
        )
        if not full_text.strip():
            return

        try:
            audio_bytes = await tts_handler.synthesize(full_text)
            if audio_bytes:
                import io
                await update.message.reply_voice(voice=io.BytesIO(audio_bytes))
        except Exception as exc:
            logger.warning("TTS send failed", error=str(exc))
```

- [ ] **Step 5: Wire TTS into `agentic_voice` (the voice-input handler, ~line 1325)**

After the call to `_handle_agentic_media_message` returns (line 1332), we need TTS. But `_handle_agentic_media_message` sends text internally. The cleanest approach: pass a flag and handle TTS at the end of `_handle_agentic_media_message`.

In `_handle_agentic_media_message` signature (~line 1342), add a parameter:

```python
        send_tts: bool = False,
```

At the end of `_handle_agentic_media_message`, after the text-sending loop (~line 1451), add:

```python
        # Send TTS voice response if requested
        if send_tts:
            await self._maybe_send_tts(update, context, formatted_messages)
```

In `agentic_voice` (~line 1325), change the call to pass `send_tts=True`:

```python
            await self._handle_agentic_media_message(
                update=update,
                context=context,
                prompt=processed_voice.prompt,
                progress_msg=progress_msg,
                user_id=user_id,
                chat=chat,
                send_tts=True,
            )
```

- [ ] **Step 6: Wire TTS into `agentic_text` (for /voice toggle, ~after line 1060)**

After the text-sending block in `agentic_text` (after the images block at ~line 1071), add:

```python
        # Send TTS if always-on toggle is active
        if context.user_data.get("tts_always_on"):
            await self._maybe_send_tts(update, context, formatted_messages)
```

- [ ] **Step 7: Commit**

```bash
cd ~/cowork/claude-code-telegram
git add src/bot/orchestrator.py
git commit -m "feat: add /voice toggle and TTS response sending" --no-gpg-sign
```

---

### Task 5: Add MINIMAX_API_KEY to .env and restart bot

**Files:**
- Modify: `~/cowork/claude-code-telegram/.env`

- [ ] **Step 1: Add TTS config to .env**

Append to `.env`:

```bash
# === TTS (Text-to-Speech) ===
TTS_ENABLED=true
TTS_PROVIDER=minimax
TTS_MODEL=speech-2.8-hd
TTS_VOICE_ID=English_Graceful_Lady
MINIMAX_API_KEY=sk-cp-dZRuzrtk3d8Esyv5CLGbK46oLfDxXoCEL-3zPkaPZerGeN8J1NieC1VAJxpfje_h9Pubbk6i0GlfcMBKMUnGnRwu3fQSvLRTJyTHbWgwSLBztOe1-OglnCE
```

- [ ] **Step 2: Restart the bot**

```bash
systemctl --user restart claude-telegram-bot
sleep 2
systemctl --user status claude-telegram-bot
```

Expected: `active (running)`, logs should show `TTS handler feature enabled`.

- [ ] **Step 3: Verify TTS in logs**

```bash
journalctl --user -u claude-telegram-bot --since "1 min ago" | grep -i tts
```

Expected: line containing `TTS handler feature enabled`.

- [ ] **Step 4: Commit .env is gitignored (verify)**

```bash
grep '.env' ~/cowork/claude-code-telegram/.gitignore
```

Expected: `.env` is listed (no commit needed for .env).

---

### Task 6: End-to-end test

- [ ] **Step 1: Test voice-in → voice+text out**

Send a voice note to the Telegram bot. Expected:
1. "Transcribing..." progress message
2. "Working..." progress message
3. Text response from Claude
4. Voice message (MP3) with the same response spoken

- [ ] **Step 2: Test /voice toggle**

Send `/voice` to the bot. Expected: `Voice responses: ON`.
Send a text message. Expected: text response + voice message.
Send `/voice` again. Expected: `Voice responses: OFF`.
Send a text message. Expected: text response only (no voice).

- [ ] **Step 3: Test long response**

Ask Claude something that produces a long response (>500 chars). Expected: full text + voice of the response (truncated to 10,000 chars if needed).

- [ ] **Step 4: Final commit with all changes**

```bash
cd ~/cowork/claude-code-telegram
git add -A
git commit -m "feat: add MiniMax TTS for voice responses" --no-gpg-sign
git push origin main
```
