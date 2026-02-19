# DroidRun Development Guide

## Project Overview

DroidRun is a framework for controlling Android and iOS devices through LLM agents using natural language commands. It enables mobile automation by combining multi-agent architectures (planning + execution) with device interaction through ADB and the Portal APK.

- **GitHub:** https://github.com/droidrun/droidrun
- **Docs:** https://docs.droidrun.ai
- **License:** MIT
- **Python:** 3.11–3.13 (3.14: use `pip install -e . --ignore-requires-python`)
- **Current Version:** 0.5.0.dev4

---

## Installation & Development Setup

```bash
# Editable install (changes take effect immediately — no reinstall needed)
pip install -e .

# Force-install on Python 3.14+
pip install -e . --ignore-requires-python

# Provider-specific extras
pip install -e ".[anthropic]"
pip install -e ".[langfuse]"
```

### Environment Variables

```bash
export ANTHROPIC_API_KEY="sk-ant-..."   # Claude
export OPENAI_API_KEY="sk-..."          # OpenAI
export GOOGLE_API_KEY="..."             # Gemini
```

---

## Directory Structure

```
droidrun/
├── agent/
│   ├── droid/droid_agent.py        # Main workflow controller (DroidAgent)
│   ├── manager/                    # Planning agent (reasoning mode)
│   ├── executor/                   # Action execution agent (reasoning mode)
│   ├── codeact/                    # FastAgent/CodeAct (direct mode)
│   ├── scripter/                   # Off-device Python code execution
│   ├── oneflows/                   # Single-purpose sub-agents (AppOpener, etc.)
│   ├── utils/
│   │   ├── actions.py              # Atomic action implementations
│   │   ├── signatures.py           # Action signatures + capability deps
│   │   └── llm_picker.py           # LLM provider loader
│   ├── tool_registry.py            # Central tool registry & dispatcher
│   └── action_context.py           # Context passed to every action
├── tools/
│   ├── driver/
│   │   ├── android.py              # AndroidDriver (ADB + Portal)
│   │   ├── ios.py                  # IOSDriver
│   │   ├── recording.py            # RecordingDriver (macro recording)
│   │   └── stealth.py              # StealthDriver wrapper
│   ├── android/portal_client.py    # Portal APK communication
│   ├── filters/                    # UI tree filtering (ConciseFilter, DetailedFilter)
│   └── formatters/indexed_formatter.py  # Flatten tree → indexed text for LLM
├── cli/
│   ├── main.py                     # Click-based CLI entry point
│   ├── event_handler.py            # Event processing
│   └── doctor.py                  # Health check command
├── config/
│   ├── prompts/                    # Jinja2 prompt templates
│   │   ├── codeact/tools_system.jinja2
│   │   ├── manager/system.jinja2
│   │   └── executor/system.jinja2
│   └── app_cards/                  # App-specific instruction cards
├── config_manager/config_manager.py  # Config dataclasses (DroidrunConfig)
├── portal.py                       # Portal APK setup & management functions
└── mcp/                            # Model Context Protocol integration
examples/
└── tv2play/                        # Example tasks for TV 2 Play testing
```

---

## Running the CLI

```bash
# Basic usage (auto-inserts "run" subcommand)
droidrun "open YouTube and search for cats"

# Explicit subcommand (required when passing flags before the task)
droidrun run --tv --provider Anthropic --model claude-sonnet-4-6 "task"

# Common flags
droidrun run "task" \
  --device emulator-5556 \
  --provider Anthropic \
  --model claude-sonnet-4-6 \
  --steps 50 \
  --vision \
  --reasoning \
  --debug \
  --tv              # Force Android TV mode

# Other commands
droidrun devices              # List connected ADB devices
droidrun setup                # Install Portal APK on device
droidrun ping                 # Check Portal connectivity
droidrun doctor               # System health check
droidrun connect 192.168.1.x:5555
```

> **Note:** `droidrun --tv "task"` does NOT work — options before the task string don't
> trigger the auto-`run` insertion. Always use `droidrun run --tv "task"`.

---

## Key Architecture: Agent Pipeline

```
User Command (CLI / DroidAgent API)
    ↓
ConfigLoader (YAML) → DroidAgent.__init__()
    ↓
AndroidDriver.connect()
  → detect TV mode (ro.build.characteristics)
  → mutate self.supported set
    ↓
ToolRegistry
  → register ATOMIC_ACTION_SIGNATURES
  → disable_unsupported(driver.supported | state_provider.supported)
    ↓
[reasoning=False]          [reasoning=True]
FastAgent / CodeActAgent   ManagerAgent → ExecutorAgent loop
    ↓                           ↓
ToolRegistry.execute(action, params, ctx)
    ↓
actions.py function (click, dpad_up, etc.)
    ↓
AndroidDriver (tap / press_key / etc.)
    ↓
ADB → Device → Portal APK
    ↓
AndroidStateProvider.get_state()
  → PortalClient.get_state()     # raw a11y tree + phone_state
  → TreeFilter                   # remove noise
  → IndexedFormatter             # flatten + index + focus markers
    ↓
Formatted UI text → LLM → next action
```

---

## Configuration System

All config is defined as Python dataclasses in `config_manager/config_manager.py`.

```yaml
# config.yaml example
agent:
  max_steps: 50
  reasoning: false        # false = direct (FastAgent), true = plan+execute
  streaming: true

device:
  serial: "192.168.1.100"
  platform: android       # or "ios"
  tv_mode: true           # None=auto-detect, true/false=explicit

llm_profiles:
  manager:
    provider: Anthropic
    model: claude-sonnet-4-6
    temperature: 0.2
  executor:
    provider: Anthropic
    model: claude-sonnet-4-6
    temperature: 0.1

tools:
  disabled_tools: ["click_at", "click_area", "long_press_at"]
  stealth: false

logging:
  debug: false
  save_trajectory: none   # none | step | action
```

### Key Dataclasses

```python
DeviceConfig:
  serial: Optional[str]       # None = auto-select first device
  use_tcp: bool               # TCP vs USB ADB
  platform: str               # "android" | "ios"
  tv_mode: Optional[bool]     # None=auto, True/False=explicit

AgentConfig:
  max_steps: int = 15
  reasoning: bool = False
  streaming: bool = True
  after_sleep_action: float = 1.0
  wait_for_stable_ui: float = 0.3
```

---

## Tools & Actions System

### Atomic Actions (`actions.py`)

Every device interaction is an `async def action(params, *, ctx: ActionContext) -> ActionResult`.

```python
# Touch (phone/tablet)
click(index)
long_press(index)
click_at(x, y)
swipe(coordinate, coordinate2, duration)
type(text, index, clear=False)

# System
system_button(button)   # "back" | "home" | "enter" | "menu" | "play_pause"
open_app(text)
wait(duration)

# D-Pad / Android TV (NEW)
dpad_up(repeat=1)
dpad_down(repeat=1)
dpad_left(repeat=1)
dpad_right(repeat=1)
dpad_center(repeat=1)   # select / OK

# State
remember(information)
complete(success, message)
```

### Capability Gating (`signatures.py`)

Each action declares `deps` — the set of capabilities it requires:

```python
"click":      deps={"tap", "element_index"}
"dpad_up":    deps={"tv_dpad"}
"swipe":      deps={"swipe", "convert_point"}
"wait":       deps={}   # always available
```

`ToolRegistry.disable_unsupported(driver.supported)` removes any tool whose deps aren't satisfied.

**On Android TV:**
- `driver.supported` loses `"tap"` and `"swipe"`, gains `"tv_dpad"`
- Result: `click`, `swipe`, `long_press` are removed; `dpad_*` are enabled

### Adding a New Action

1. Implement in `actions.py`:
   ```python
   async def my_action(param: str, *, ctx: ActionContext) -> ActionResult:
       try:
           await ctx.driver.press_key(99)
           return ActionResult(success=True, summary="Done")
       except Exception as e:
           return ActionResult(success=False, summary=str(e))
   ```

2. Register in `signatures.py`:
   ```python
   "my_action": {
       "parameters": {"param": {"type": "string", "required": True}},
       "description": "Does something...",
       "function": my_action,
       "deps": {"press_key"},
   }
   ```

---

## Prompt Templates

Jinja2 templates in `config/prompts/{agent}/system.jinja2`.

**Variables available in all templates:**
```jinja2
{{ instruction }}          {# user goal #}
{{ tool_descriptions }}    {# formatted tool list #}
{{ variables.tv_mode }}    {# True on Android TV #}
{{ variables.vision }}     {# True if screenshots enabled #}
{{ available_tools }}      {# list of enabled tool names #}
```

**TV mode conditional blocks** appear in all 3 system prompts:
```jinja2
{% if variables and variables.tv_mode %}
  ... TV-specific instructions ...
{% endif %}
```

---

## UI Tree Flow

```
Portal APK (/state_full endpoint)
  → { a11y_tree, phone_state, device_context }
      ↓
TreeFilter (ConciseFilter or DetailedFilter)
  → remove invisible/out-of-bounds nodes, preserve isFocused/isAccessibilityFocused
      ↓
IndexedFormatter._format_node()
  → assigns index, extracts text/className/bounds
  → isFocused = node.isFocused OR node.isAccessibilityFocused   ← TV fix
      ↓
IndexedFormatter.format()
  → finds focused_index (tree scan → phone_state text/resourceId fallback)
  → renders "[FOCUSED]" markers and "Focused Element: index N — 'text'" header
      ↓
Formatted text → LLM prompt
```

**Example output:**
```
**Current Phone State:**
• App: Netflix (com.netflix.ninja)
• Keyboard: Hidden
• Focused Element: index 3 — 'Stranger Things'

1. LinearLayout: "row_card", "Trending Now" - (0,0,1920,60)
2. ImageView: "thumbnail", "Breaking Bad" - (20,80,340,260)
3. [FOCUSED] ImageView: "thumbnail", "Stranger Things" - (360,80,680,260)
4. ImageView: "thumbnail", "The Crown" - (700,80,1020,260)
```

---

## Android TV / D-Pad Feature

### Files Changed

| File | Change |
|------|--------|
| `config_manager/config_manager.py` | `DeviceConfig.tv_mode: Optional[bool]` |
| `tools/driver/android.py` | Per-instance `supported` set; `tv_mode` param; `_detect_tv_mode()`; overlay hide |
| `agent/utils/actions.py` | `dpad_up/down/left/right/center`; extended `system_button` |
| `agent/utils/signatures.py` | 5 dpad entries with `deps={"tv_dpad"}` |
| `agent/droid/droid_agent.py` | Pass `tv_mode` to driver; inject into `custom_variables` |
| `cli/main.py` | `--tv/--no-tv` flag |
| `config/prompts/*/system.jinja2` | TV instruction blocks |
| `tools/formatters/indexed_formatter.py` | `isAccessibilityFocused` support; `[FOCUSED]`/`[SELECTED]` markers; focused index in header |

### Detection

```bash
adb shell getprop ro.build.characteristics
# Returns "tv,hdmi" on Android TV → tv_mode = True
# Returns "default,phone" on phone → tv_mode = False
```

### ADB Keycodes Used

```
DPAD_UP     = 19
DPAD_DOWN   = 20
DPAD_LEFT   = 21
DPAD_RIGHT  = 22
DPAD_CENTER = 23   # select / OK
MENU        = 82
PLAY_PAUSE  = 85
```

---

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `llama-index` | LLM workflows, agent framework |
| `llama-index-llms-*` | Provider integrations (Anthropic, OpenAI, Google, etc.) |
| `async_adbutils` | Async ADB communication |
| `pydantic>=2` | Data validation |
| `rich` | Terminal output |
| `click` | CLI framework |
| `httpx` | HTTP client (Portal TCP) |
| `textual` | TUI framework |
| `mcp` | Model Context Protocol |
| `arize-phoenix` | Tracing backend |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `PackageNotFoundError: droidrun` | `pip install -e .` |
| `requires Python <3.14` | `pip install -e . --ignore-requires-python` |
| `--tv` not recognized | Use `droidrun run --tv "task"`, not `droidrun --tv "task"` |
| Auth error (Anthropic) | Check `$ANTHROPIC_API_KEY` has no leading space |
| Focused Element always "none" | Portal may not emit `isAccessibilityFocused`; fallback matches by text |
| Portal overlay visible on TV | `toggle_overlay()` called on connect — check Portal version ≥ 0.4.1 |
| `AnthropicCompletionResponse` Pydantic error | Patched in `llama_index/llms/anthropic/base.py`: `text=chat_response.message.content or ""` |

---

## Fork & Branch

- **Upstream:** `https://github.com/droidrun/droidrun` (remote: `origin`)
- **Fork:** `https://github.com/ArnesenTV2/droidrun` (remote: `fork`)
- **Feature branch:** `feature/android-tv-dpad-support`

```bash
# Push changes to fork
git push fork feature/android-tv-dpad-support

# Open PR to upstream when ready
gh pr create --repo droidrun/droidrun --head ArnesenTV2:feature/android-tv-dpad-support
```
