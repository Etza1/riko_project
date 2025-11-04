## Quick context for code-writing agents

This repository (Project Riko) is a small voice-based conversational pipeline combining:
- OpenAI (LLM) via the official Python client
- Faster-Whisper for ASR
- GPT-SoVITS (HTTP API) for TTS

Primary runtime is Python 3.10 (Linux/Windows tested). The Python package entry points and runtime scripts live under the `riko_project/` folder.

Core files to inspect before making changes
- `riko_project/server/main_chat.py` — main loop: records via ASR, calls the LLM, calls TTS, plays audio, and deletes temporary WAV files.
- `riko_project/server/process/asr_func/asr_push_to_talk.py` — push-to-talk recorder using `sounddevice`; waits for ENTER to start/stop.
- `riko_project/server/process/llm_funcs/llm_scr.py` — OpenAI Responses API usage; reads `character_config.yaml` for API key, model, and history file.
- `riko_project/server/process/tts_func/sovits_ping.py` — posts text to local GPT-SoVITS HTTP server at `http://127.0.0.1:9880/tts` and writes returned binary audio to disk.
- `riko_project/character_config.yaml` — central config (OPENAI_API_KEY, model, history_file, sovits_ping_config). Edit this to change persona, model, or file paths.

Important architecture & data flow notes
- Flow: microphone -> save WAV -> Faster-Whisper transcribe -> llm_scr.llm_response(...) -> sovits_ping.sovits_gen(...) -> play back WAV
- LLM history: conversation history is appended to `history_file` from `character_config.yaml`. The LLM client persists history as JSON; edits should respect that format.
- TTS API: the TTS generator expects a running local API (GPT-SoVITS) on port 9880. The code does not start this server — it must be started separately.

Developer workflows and commands (project-specific)
- Install deps (script): `riko_project/install_reqs.sh` — run this in the `riko_project/` directory. It uses `pip`/`uv` and installs `extra-req.txt` and `requirements.txt`.
- Start TTS server: this repo expects a GPT-SoVITS HTTP server available at `http://127.0.0.1:9880/tts`. (If you don't have it, follow upstream GPT-SoVITS setup.)
- Run the main loop: from `riko_project/` run `python server/main_chat.py` — it will instantiate `WhisperModel` and then prompt for ENTER to record.

Project-specific conventions and gotchas (do not change these without careful checks)
- Config filename is `character_config.yaml` (README sometimes mentions `config.yaml` — use `character_config.yaml`).
- The push-to-talk recorder uses blocking `input()` to start/stop recording. Many edits that introduce async behavior or non-blocking input will change runtime characteristics.
- `sovits_ping.sovits_gen()` POSTS JSON and writes response.content directly to disk; if the TTS server returns errors, the function prints and returns `None` — callers do not always handle `None` robustly.
- `main_chat.py` deletes all `.wav` files in `audio/` after playback. If you need persistent audio for debugging, comment out that cleanup.

Integration points and external dependencies
- OpenAI: `riko_project/server/process/llm_funcs/llm_scr.py` expects `OPENAI_API_KEY` in `character_config.yaml`. It uses the `openai.OpenAI` client and the Responses API.
- Faster-Whisper: code constructs `WhisperModel("base.en", device="cpu")` in `main_chat.py` — model name and device are explicit; tests use CPU by default.
- GPT-SoVITS: TTS server must serve `/tts` and return raw WAV bytes. Configurable fields are in `sovits_ping_config` in `character_config.yaml`.

Examples to reference in edits
- To change persona text, edit the system prompt under `presets.default.system_prompt` in `character_config.yaml`.
- To change ASR model/device: edit the `WhisperModel(...)` call in `server/main_chat.py`.
- To debug TTS failures: run `riko_project/server/process/tts_func/sovits_ping.py` as a script — it prints elapsed time and errors.

Testing and quick debug tips
- If audio playback fails, check `sounddevice` availability and system audio devices.
- If transcriptions are empty, ensure `faster-whisper` is installed and the model path/name is valid.
- If LLM calls fail, confirm `OPENAI_API_KEY` and model string in `character_config.yaml`.

When editing: keep changes minimal and respect existing file-level side effects (blocking input(), file I/O, external HTTP calls). Mention any breaking changes in PR description.

If anything in this file is unclear or you want additional examples (unit tests for `llm_scr` or a minimal integration test for `sovits_ping`), tell me which area to expand.
