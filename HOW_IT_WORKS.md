# Keyboard Layout Watchdog — How It Works

A Windows tool that fixes typing in the wrong keyboard layout (Thai
Kedmanee ↔ US English).

- Press **F7** — convert the text you have highlighted (mouse-drag or
  Shift+arrow).
- Press **F8** — convert the highlighted text, or if nothing is
  highlighted, automatically grab the last word you typed.

---

## The Big Idea

Imagine you wanted to type `สวัสดี` but forgot to switch to Thai layout.
You typed `l;ylfu` in English layout, and the screen shows `l;ylfu`.

Each key on your keyboard produces a **different character** depending on
the active layout:

| Physical Key | English (QWERTY) | Thai (Kedmanee) |
|--------------|------------------|------------------|
| L            | l                | ส               |
| ;            | ;                | ว               |
| Y            | y                | ั               |
| L            | l                | ส               |
| F            | f                | ด               |
| U            | u                | ี               |

So `l;ylfu` typed on English **is the same physical key sequence** as
`สวัสดี` typed on Thai. We just translate one to the other.

---

## File-by-File Walkthrough

### 1. `keymap.py` — The Translation Table

This is the dictionary that knows "if you press the L key, you get `l` on
English and `ส` on Thai".

```python
QWERTY_TO_THAI = {
    "l": "ส",
    ";": "ว",
    "y": "ั",
    "f": "ด",
    "u": "ี",
    ...
}
```

**Two functions:**

- `remap_to_thai(text)` — takes English chars, returns the Thai chars that
  the same physical keys would have produced.
  - Example: `remap_to_thai("l;ylfu")` → `"สวัสดี"`

- `remap_to_qwerty(text)` — the reverse. Takes Thai chars, returns English.
  - Example: `remap_to_qwerty("้ำสสน")` → `"hello"`

The reverse map (`THAI_TO_QWERTY`) is auto-built by inverting
`QWERTY_TO_THAI`. We checked it has no collisions.

---

### 2. `swapper.py` — The Layout Switcher

Talks to Windows directly using `ctypes` (Python's interface to OS APIs)
to switch the keyboard layout.

**Key function:**

```python
def switch_layout(target):
    """target = 'th' or 'en'"""
```

How it works internally:
1. `LoadKeyboardLayoutW(klid, KLF_ACTIVATE)` — tells Windows "make sure
   the Thai/English layout is loaded and ready to use".
2. `PostMessageW(hwnd, WM_INPUTLANGCHANGEREQUEST, 0, hkl)` — sends a
   message to the focused window: "switch input language now".

The constants `LANG_TH = 0x041E` and `LANG_EN_US = 0x0409` are Windows'
internal language IDs.

**Note:** We don't ask Windows for the current layout — that turned out
to be unreliable on this machine. We detect the layout from the typed
text itself (see below).

---

### 3. `main.py` — The Core Logic

This is where everything comes together. The whole thing runs in two parts:

#### Setup (the `main()` function)

```python
keyboard.add_hotkey("f7", _convert_selection_only, suppress=False)
keyboard.add_hotkey("f8", _convert_smart,           suppress=False)
keyboard.wait("ctrl+alt+q")
```

- Registers F7 / F8 with their handlers.
- `suppress=False` is intentional — `suppress=True` silently fails on
  some Windows setups; without it the F-keys still reach other apps but
  the hotkey actually fires.
- Then waits forever until you press Ctrl+Alt+Q to quit.

#### F7 vs F8

| Key | Handler                     | Behavior                                          |
|-----|-----------------------------|---------------------------------------------------|
| F7  | `_convert_selection_only`   | Convert what you've highlighted; do nothing if no selection |
| F8  | `_convert_smart`            | Convert highlighted text, or auto-grab the last word |

Both eventually call the shared `_do_convert(full_sel, word)` which does
the actual layout detection, remapping, and paste.

#### What `_do_convert()` does

**Step 1: Figure out what to convert**

```python
existing = _grab_selection()
if existing.strip():
    full_sel, word = existing, existing
else:
    full_sel, word = _grab_previous_run()
```

If you already selected text with mouse/keyboard, use that. Otherwise,
grab the previous "word" automatically.

**Step 2: How `_grab_previous_run()` selects the word**

This is the trickiest part. The challenge: Notepad treats `;` as a word
boundary, but `;` is part of Thai words when typed on English layout.

So we **over-select**: send `Ctrl+Shift+Left` ten times in a row.

```python
for _ in range(10):
    keyboard.send("ctrl+shift+left")
    time.sleep(0.015)
```

This creates a big selection that crosses any punctuation. Then we
copy it via clipboard and look at it:

```python
sel = _grab_selection_raw()  # = "previous text\nll;ylfu"
ws_indices = [i for i, c in enumerate(sel) if c.isspace()]
if ws_indices:
    last_ws = ws_indices[-1]
    return sel, sel[last_ws + 1:]   # full_sel, word
```

If the selection contains whitespace, the **rightmost token** (after the
last whitespace) is the word we want to convert. The "full_sel" is the
whole thing we selected — we'll use it in step 4.

**Step 3: Detect which layout produced the text**

```python
def detect_layout_from_text(text):
    for ch in text:
        if 0x0E00 <= ord(ch) <= 0x0E7F:  # Thai Unicode block
            return "th"
    return "en"
```

If the captured text contains any Thai character → user was on Thai
layout (and meant English). Otherwise → user was on English (and meant
Thai). This is bulletproof because it's based on what's actually on
screen.

**Step 4: Build the replacement text**

```python
prefix = full_sel[: len(full_sel) - len(word)] if full_sel != word else ""
paste_text = prefix + converted
```

The selection might have grabbed too much (some extra text before the
word we care about). We **paste back the prefix unchanged** and only
convert the target word. So the over-selection causes no harm.

**Step 5: Switch layout and paste**

```python
switch_layout(target)
time.sleep(0.08)
pyperclip.copy(paste_text)
keyboard.send("ctrl+v")
```

We change the OS keyboard layout (so future typing goes to Thai/English
correctly), then paste the corrected text via clipboard. Clipboard paste
handles Unicode reliably across all Windows apps.

---

## Full Flow Example

You're on English layout. You type `Hello l;ylfu` and press F8.

1. **`_grab_selection()`** → returns `""` (nothing selected).
2. **`_grab_previous_run()`** sends `Ctrl+Shift+Left` × 10. Selection
   becomes `Hello l;ylfu`. Returns `("Hello l;ylfu", "l;ylfu")`.
3. **`detect_layout_from_text("l;ylfu")`** → `"en"` (no Thai chars).
4. **`remap_to_thai("l;ylfu")`** → `"สวัสดี"`.
5. **prefix** = `"Hello "` (the part before our target word).
6. **`paste_text`** = `"Hello " + "สวัสดี"` = `"Hello สวัสดี"`.
7. **`switch_layout("th")`** — Windows now on Thai layout.
8. **paste** — selection is replaced with `Hello สวัสดี`.

Done. You can keep typing in Thai now.

---

## What Each Sleep Is For

You'll see `time.sleep(0.05)` and similar sprinkled around. These are
*not* arbitrary — they exist because Windows is asynchronous:

- After `Ctrl+Shift+Left`, Windows needs a moment to update the selection
  before we can read it.
- After `pyperclip.copy("")` + `Ctrl+C`, Windows needs time to actually
  put the new selection on the clipboard before `pyperclip.paste()` will
  see it.
- After `switch_layout()`, the layout change is queued — we wait so the
  paste happens *after* the switch.

Without these sleeps the tool randomly fails because operations race.

---

## Running It

1. **Install dependencies** (one time):
   ```
   pip install -r requirements.txt
   ```
2. **Run as Administrator** (the `keyboard` library needs admin to install
   global hooks):
   ```
   python main.py
   ```
3. Type wrong-layout text anywhere → press **F8** → fixed.
4. Press **Ctrl+Alt+Q** to quit.
