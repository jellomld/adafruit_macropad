# Project Agents.md Guide for the **Adafruit MacroPad** Codebase

This `AGENTS.md` file gives OpenAI Codex (or any other code‑generating agent) all of the context it needs to work safely and productively in **[`jellomld/adafruit_macropad`](https://github.com/jellomld/adafruit_macropad)**.

---

## 1  Repository Map

| Path                                         | Purpose                                                                                                                                                                                              |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/apps`                                      | High‑level *App* classes that drive individual macro sets, games, or utilities. Each file exports a subclass of `BaseApp`.                                                                           |
| `/lib`                                       | Third‑party CircuitPython libraries copied from Adafruit’s bundle. **Do not edit** unless you are upgrading a library version.                                                                       |
| `/utils`                                     | Re‑usable helpers such as `constants.py`, key‑code builders, timer utilities, etc.                                                                                                                   |
| `code.py`                                    | Device entry‑point. Instantiates `AppPad` and loads the default or user‑supplied app. **Must remain in the root and keep the same name** so the MacroPad’s bootloader can find it.                   |
| `default_settings.py`                        | Declarative defaults (device brightness, default OS, inactivity timeout, etc.).  Agents should **read** but *not* hard‑code these values.                                                            |
| `.github/workflows`                          | Continuous‑integration jobs (Black format, isort, lint, and unit‑tests).                                                                                                                             |
| `pyproject.toml` / `.pre-commit-config.yaml` | Tooling config for **Black**, **isort**, and **pre‑commit**. All formatting/lint fixes *must* pass these checks before a PR can merge. ([github.com](https://github.com/jellomld/adafruit_macropad)) |

*The full directory listing can be seen in the GitHub UI.* ([github.com](https://github.com/jellomld/adafruit_macropad))

---

## 2  Coding Conventions

### 2.1  Language & Style

* **CircuitPython 8 / CPython 3.9 syntax** only—it must run on the RP2040.
* Auto‑format **with Black 23.x** and **isort 5** (respect `pyproject.toml`).
* Keep lines ≤ 100 chars (screens within Mu / Thonny wrap awkwardly).
* Prefer **dataclasses** for simple state containers (they are available in CP 8).
* Use `typing` *sparingly*—type hints help but increase file size.
* Functions interacting with USB HID/KM send **must include a short docstring** explaining the intended keystroke sequence.
* Never allocate large lists/dicts inside tight loops—flash/RAM is limited.

### 2.2  Hardware APIs

| Action                  | Library / API                              |
| ----------------------- | ------------------------------------------ |
| Key‑matrix scanning     | `adafruit_matrixkeypad`                    |
| NeoPixel under‑key LEDs | `adafruit_dotstar`                         |
| Rotary encoder          | `adafruit_rotaryio` + `adafruit_debouncer` |
| USB HID                 | `adafruit_hid`                             |

Agents **must** reuse the wrappers already present under `/utils` instead of importing Adafruit libs directly; this keeps ISR timing predictable.

### 2.3  File & Symbol Naming

* **Snake‑case** for files & identifiers (`window_manager_app.py`).
* Constants in `UPPER_SNAKE` live in `utils/constants.py`.
* App classes in `/apps` must be `PascalCase` and extend `BaseApp`.

---

## 3  Extending the MacroPad

### 3.1  Adding a New Macro App

1. Create `apps/<name>_app.py`.
2. Define `class <Name>App(BaseApp):` and implement:

   * `display_name` → shown on OLED header
   * `key_layout()` → returns the list of `KeyDef` objects
   * `on_rotate`, `on_press`, etc. as needed.
3. Update `<name>_app` in `apps/__init__.py` **or** expose it via a *switcher* key in an existing app.
4. Verify RAM usage (`import gc; gc.mem_free()`). Stay below **20 kB free** after load.

### 3.2  User‑specific Customisation

End‑users can drop a `user.py` or `user/__init__.py` next to `code.py` that exports `DEFAULT_APP(macropad) → BaseApp`. This file is git‑ignored, so user configs survive updates. The preferred flow is described in the README. ([github.com](https://github.com/jellomld/adafruit_macropad))

---

## 4  Testing & Checks

Although the embedded target can’t run `pytest`, we **mirror** host‑side tests under `/tests` to exercise non‑hardware logic (e.g. key‑sequence builders, timers). When adding logic that *can* be unit‑tested:

```bash
# Run all host‑side tests
pytest
```

CI will automatically execute:

```bash
# Static analysis / formatting
pre-commit run --all-files
```

A pull request **must not** break the CI matrix.

---

## 5  Pull‑Request Checklist

Before requesting review, ensure your agent:

1. Keeps the PR focused on a single topic (one new app, a refactor, etc.).
2. Includes a descriptive title and body linking any issues.
3. Passes *all* GitHub Actions.
4. Adds/updates inline documentation where behaviour changed.
5. Provides a screenshot or short video **if** the OLED UI/LEDs were modified.

---

## 6  Deployment to Device

1. Re‑build the `.uf2` *only* if core libraries changed.
2. Otherwise, copy changed **.py** files to the mounted `CIRCUITPY` drive.
3. **Eject** the drive to flush caches; the MacroPad will soft‑reboot.

---

## 7  Gotchas & Best‑Practices

* Keep interrupt service routines (ISRs) tiny—never allocate in an ISR.
* Debounce every key/encoder press with `utils.debounce_factory`.
* Use `supervisor.reload()` instead of `microcontroller.reset()` to retain USB.
* `audioio` is disabled to save RAM—avoid importing it.
* Save power: after 60 s idle, call `macropad.leds_off()` unless the active app overrides it.

---

> **Happy hacking!**  May your macros be fast and your LEDs bright.
