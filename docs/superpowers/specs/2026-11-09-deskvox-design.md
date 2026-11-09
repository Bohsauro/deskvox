# deskvox — design

## Intent

A Google-Assistant-style voice assistant for the desktop. The user presses a hotkey, speaks, and an LLM (local or online) answers and performs actions: read the news, look something up on the web, open a program.

Built first for the author's own use on Windows in Italian, published as an open-source GitHub project whose defaults are English. Same codebase should run on Windows, macOS and Linux, with an easy installer for each.

## Goals

- Global hotkey triggers listening from anywhere.
- Speech-to-text runs locally (Whisper). Language configurable; default `en`, author uses `it`.
- Any OpenAI-compatible chat backend: OpenRouter (e.g. free Nemotron), Ollama, LM Studio. Switching is a settings change (base URL, model, API key).
- The LLM can call tools: news headlines, web search, open an application, open a URL.
- An always-on-top panel anchored to the right edge of the screen shows the conversation.
- Reply mode is a setting: **text only** or **voice + text**.
- One-click installer per OS: Windows setup `.exe`, macOS `.dmg`, Linux AppImage.

## Non-goals (v1)

- Wake word ("hey deskvox"). Hotkey only.
- Smart-home control, calendar, email, account integrations.
- Multi-turn memory across app restarts (conversation resets on restart).
- Mobile.
- Code signing / notarization (installers will trigger SmartScreen / Gatekeeper warnings; documented in README).

## Anti-goals

- Requires an account or cloud service to work at all. Fully local setup (Ollama + local Whisper + Piper) must work offline except for tools that need the internet.
- Sends audio to a cloud service. Audio never leaves the machine.
- LLM can run arbitrary shell commands. Tools are a fixed, narrow list.
- Panel steals focus or blocks the user's work when not in use.

## Constraints

- Python 3.11+, PySide6 for UI.
- OpenRouter `:free` models are rate-limited and not all support tool calling. The default suggested model must support tools; when a model rejects tools, show a clear error in the panel.
- No free Google Search API. Web search uses DuckDuckGo (`ddgs` package); fallback action opens the browser on a Google search URL.
- PyInstaller cannot cross-compile: each OS builds its own installer on GitHub Actions.
- macOS needs Accessibility + Microphone permissions for global hotkey and recording. Linux Wayland limits global hotkeys and always-on-top; X11 is the supported Linux target, Wayland is best-effort.

## Approaches considered

1. **All Python (PySide6 + faster-whisper + Piper + OpenAI client)** — chosen. Most mature voice/AI ecosystem, single language, simplest for contributors. Cost: large bundle (~200 MB+).
2. Tauri/Electron UI + Python sidecar — nicer UI, but two processes and IPC double the complexity.
3. C#/WPF native — lighter install, but weaker AI/voice libraries and Windows-only.

User picked 1.

## Design

### Components (one module each under `src/deskvox/`)

| Module | Job |
|---|---|
| `app.py` | Entry point. Wires everything, owns the Qt event loop and the tray icon. |
| `config.py` | Load/save settings as JSON in the per-user config dir (`platformdirs`). Validates values; falls back to defaults on a broken file and tells the user. |
| `hotkey.py` | Global hotkey via `pynput`. Default: hold `Ctrl+Alt+Space` to talk, release to stop (push-to-talk). Configurable combo. |
| `audio.py` | Record mic with `sounddevice` at 16 kHz mono while the hotkey is held. Cap at 60 s. |
| `stt.py` | `faster-whisper` transcription. Model size setting (default `small`; `base` for slow PCs). Model downloaded on first run with progress shown in the panel. |
| `llm.py` | `openai` client with configurable `base_url`, `api_key`, `model`. Runs the tool-calling loop (max 5 tool rounds per request). System prompt tells the model to reply in the configured language and be brief. |
| `tools/` | One file per tool, each exposing a JSON schema + a function. Registry lists them for the LLM. |
| `tts.py` | Local voice output when mode is voice + text, using Piper voice models run through `sherpa-onnx` (Apache-2.0) so the app stays MIT — the maintained Piper package is GPL-3. Voice model per language, downloaded on first use. Speaking stops when the hotkey is pressed again. |
| `ui/panel.py` | Frameless, always-on-top, right-edge panel: chat bubbles, status line (listening / transcribing / thinking / speaking), text input box as fallback for typing. Slides in on hotkey; auto-hides after N seconds of inactivity (setting, default 15 s). Does not take focus when shown by hotkey. |
| `ui/settings.py` | Settings dialog: backend URL/model/key, STT language + model size, reply mode, action mode (ask/free), voice, hotkey, auto-hide delay. |
| `platform/` | OS-specific bits behind one small interface: `open_app(name)`, `list_apps()`. Windows: Start Menu `.lnk` + `os.startfile`. macOS: `/Applications` + `open -a`. Linux: `.desktop` files + `gtk-launch`/`xdg-open`. |

Heavy work (recording, STT, LLM, TTS) runs on worker threads; UI updates go through Qt signals.

### Tools (v1)

| Tool | Behavior |
|---|---|
| `get_news(topic?, count?)` | Google News RSS for the configured language/region, parsed with `feedparser`. Returns titles + sources. |
| `web_search(query)` | DuckDuckGo text results (top 5 snippets) for the LLM to answer from. |
| `open_url(url)` | Opens in default browser. Only `http`/`https`. |
| `open_app(name)` | Fuzzy-matches `name` against installed apps from `platform.list_apps()`; launches best match above a threshold, otherwise reports the closest candidates. Never runs raw commands. |
| `get_datetime()` | Current local date/time. |

### Flow

1. Hotkey down → panel slides in, status "listening", recording starts.
2. Hotkey up → status "transcribing" → text shown as user bubble.
3. Text + recent conversation (last 10 turns) sent to LLM with tool schemas → tool calls executed → final answer shown as assistant bubble.
4. If mode is voice + text → Piper speaks the answer.
5. Panel auto-hides after inactivity.

### Errors

Every failure lands in the panel as a readable message, never a silent drop: no mic, empty transcription, backend unreachable, 401 bad key, 429 rate limit (OpenRouter free), model doesn't support tools, tool failure. Details logged to a rotating log file in the user data dir.

### Secrets

API key stored in the OS keychain via `keyring`, not in the JSON config. Never logged.

### Packaging

- PyInstaller one-folder build per OS.
- Windows: Inno Setup → `deskvox-setup-<ver>.exe` (Start Menu entry, optional autostart).
- macOS: `.app` in `.dmg`, with Info.plist mic usage string.
- Linux: AppImage.
- GitHub Actions matrix (windows/macos/ubuntu) builds all three on a `v*` tag and attaches them to the GitHub Release.
- Whisper and Piper models are not bundled; downloaded on first run.

### Testing

- Unit tests (pytest) for config, tool registry, each tool (network mocked), LLM tool loop (fake client), app-name fuzzy matching.
- Manual checklist for hotkey, mic, panel and installers per OS.

## User decisions after builder review

- **Action confirmation is a setting** ("Action mode"):
  - `ask` (default): `open_app` and `open_url` show an inline confirm in the panel ("Open X? ✓ ✗") and run only on ✓. The confirm does not steal focus; it times out as "cancelled" after 30 s.
  - `free`: actions run immediately. Settings shows a one-line warning that web/news text could steer the model into opening links or apps.
- **UI language: English only.** The LLM still replies in the configured language; only app strings (buttons, statuses, errors) are English.

## Calls made after builder review (author's proposals, not user decisions)

- **CPU only** for faster-whisper in the installers (`int8` compute). CUDA adds >1 GB and extra build variants. Users running from source can enable GPU via a setting.
- **macOS: Apple Silicon only** (one `macos-14` job, one `.dmg`). Intel Macs are out of support.
- **First run:** settings open automatically with a short "choose your backend" section (OpenRouter + key, or Ollama on localhost). The hotkey shows "set up a backend first" until a backend responds to a test call.
- **News region** derives from the STT language (`it` → IT, `en` → US).
- **History turn** = one user message + the assistant's final reply; tool messages are not kept across turns.
- **Hotkey while speaking:** stops speech and starts a new recording.

## Left to the builder

- Exact visual styling of the panel (dark/light following OS theme is the default).
- Default OpenRouter model id: pick a free model that supports tool calling at build time and verify it.
- Piper voice choices per language.
- Repo layout details beyond `src/deskvox/`, `tests/`, `packaging/`, `.github/workflows/`.
