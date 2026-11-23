# Atreus Multilingual Layout — Design Notes

**Status:** Design discussion only. No changes made to firmware, host layouts, or the
`archive/` contents. Recorded 2026-09-04 so the reasoning survives the research pause.

---

## 1. Requirements and priority order

Stated priority, highest first:

1. **English / programming** — the dominant daily use. ASCII and symbol access must be
   optimal and must never regress.
2. **Norwegian** (`æ ø å`) — frequent; work language.
3. **Polish** (`ą ć ę ł ń ó ś ź ż`) — occasional; native language.

Hardware in use:

- **Keyboardio Atreus**, running Kaleidoscope firmware built from source. Chrysalis was
  tried briefly and abandoned.
- **MacBook with a physically Norwegian (ISO) built-in keyboard.** Used on the couch and
  on the train — i.e. regularly, not rarely. This matters (see §7).

History: firmware was flashed while on Linux; later moved to Windows (where the `.klc`
layout was authored); now primarily on macOS.

---

## 2. Inventory of `archive/`

| File | What it actually is |
|---|---|
| `us_warsz.txt` | An **unmodified upstream `xkeyboard-config` `symbols/us`** file — all ~40 stock variants (basic, intl, dvorak, colemak, carpalx, …). No `warsz` variant is defined in it. **No Polish characters anywhere.** Reference material, not a layout. |
| `Atreus-layout.klc` | The real work: an **MSKLC source** derived from *United States-International*. Dead keys on ``^ ' " ` ~``. AltGr plane = stock US-Intl **plus** hand-added Norwegian. **No Polish coverage at all.** |
| `KeyboardVerify.log` | Six warnings, all one cause: `æ`/`ø`/`å` were added on `e`/`o`/`a` while US-Intl already carried them on `z`/`l`/`w`. Duplicates, not errors. |
| `*.msi`, `{i386,amd64,ia64,wow64}/Atreus.dll` | Compiled output of the `.klc`. Windows only. |

Notable details in the `.klc` worth preserving:

- `Cap` column value `5` on letter keys — makes Caps Lock apply to the AltGr plane too.
  Correct, and must be repeated for the Polish letters.
- Shiftstates `6` (Ctrl+Alt) and `7` (Shift+Ctrl+Alt) are defined — the AltGr and
  Shift+AltGr planes. Needed for capitals.
- `0027@` and `0022@` — apostrophe and quote **are dead keys**. To be removed (§5).

## 2b. Current macOS state (read from the machine)

- Enabled input sources: **`Norwegian` (ID 12)** and **`USInternational-PC` (ID 15000)**.
  Switched manually depending on which keyboard is in use. This is the stated annoyance.
- `~/Library/Keyboard Layouts/` is **empty** — no custom layout has ever been installed.
  That directory is per-user and needs **no admin rights**.
- `USInternational-PC` carries dead keys — a second, previously unnamed irritation for
  writing code.

---

## 3. Keyboardio Atreus hardware facts

Verified from `Atreus2.h` and the stock `Atreus.ino` in the Kaleidoscope repo.

- Matrix is **4 rows x 12 columns**, but **columns 5 and 6 exist only on rows 2 and 3**.
  Total = 10 + 10 + 12 + 12 = **44 keys**. `PER_KEY_DATA` stubs the four absent positions
  as `XXX`.
- Stock base layer by `KeyAddr(row, col)`:

| | C0 | C1 | C2 | C3 | C4 | C5 | C6 | C7 | C8 | C9 | C10 | C11 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **R0** | Q | W | E | R | T | — | — | Y | U | I | O | **P** |
| **R1** | A | S | D | F | G | — | — | H | J | K | L | **;** |
| **R2** | Z | X | C | V | B | `` ` `` | `\` | N | M | , | . | / |
| **R3** | Esc | Tab | Gui | Shift | Bksp | Ctrl | **Alt** | Space | Fn | - | **'** | Enter |

- Left thumbs: R3C4 / R3C5. Right thumbs: R3C6 / R3C7. Fn at R3C8 is thumb-reachable.
- **All 44 base slots are occupied. There is no free key.**
- `[ ] { }` are **Fun-layer only** (`FUN` R2C0/R2C1). `p`, `;`, `'` *are* on the base layer.
- **There is no `Key_RightAlt` anywhere in the stock keymap.** The only Alt is
  `Key_LeftAlt` at R3C6. AltGr does not exist on this board by default.
- `RALT(k)` and `Key_RightAlt` both exist in `key_defs.h` (lines 303 and 247) — so adding
  AltGr is purely a keymap question.
- **Already enabled in the stock sketch:** `Qukeys`, `OneShot`, `OneShotConfig`,
  `EscapeOneShot`, `SpaceCadet`, `Macros`, `DynamicMacros`, `MouseKeys`. No new plugin
  includes are needed for the planned design.
- `OSM(mod)` accepts right-side variants, so **`OSM(RightAlt)` is legal**.
- Qukeys accepts `ShiftToLayer()` as a qukey's alternate action (per its README).
- MCU is **ATmega32U4** (Caterina bootloader): ~32 KB flash, ~28 KB usable. The stock
  plugin set is already substantial — this is a hard budget constraint on anything new.

### Gotcha: EEPROM overrides the compiled keymap

The stock sketch calls `EEPROMKeymap.setup(9)` and `Layer.move(EEPROMSettings.default_layer())`.
If Chrysalis ever wrote a keymap to EEPROM, **it wins over the keymap compiled into the
sketch** — you flash a change and see the old layout. Chrysalis was used briefly on this
board, so this is a live risk. Fix: clear EEPROM, or drop `EEPROMKeymap` from the plugin list.

---

## 4. The constraints that decide everything

### 4.1 Firmware cannot invent characters

A USB keyboard sends HID usage codes, not characters. The **host layout** turns code into
character. With a stock US layout loaded, **no keycode the Atreus can send produces `ø`.**

Therefore a host-side component is unavoidable. The only alternatives are:

- **Unicode input methods** (`Kaleidoscope-Unicode`: `Ctrl+Shift+U` on ibus, Option+hex on
  macOS *Unicode Hex Input*, Alt+numpad on Windows). Still needs per-OS setup *and* a
  firmware mode switch, is slow, and fails in Electron/Java/terminal apps and over RDP.
  Also may not fit in the remaining AVR flash.
- **Compose sequences** (native on Linux, WinCompose on Windows, third-party on macOS).

Both are fallbacks for machines where nothing can be installed. Neither is the primary
mechanism.

### 4.2 macOS applies ONE input source to ALL attached keyboards

There is no per-device layout on macOS. Both keyboards send the same HID code for the same
physical position — HID `0x33` is the `;` key on the Atreus and the `ø` key on the MacBook.
One layout gives that code one meaning.

**Consequence: one of the two keyboards' legends must lie. This is not solvable, only
choosable.**

(Linux/X11 *does* support per-device layouts via `xinput` + `setxkbmap -device`. Windows
has no native equivalent. macOS needs Karabiner-Elements.)

### 4.3 A key whose base character consumes a modifier cannot carry more modifiers

This is the decisive objection to "Norwegian host + US-compensating firmware". On macOS
Norwegian, `[` is `⌥8`. Firmware can send `LALT(Key_8)` and you get a `[`. But:

| You press | Firmware sends | Result |
|---|---|---|
| `[` | `⌥8` | `[` — works |
| `Cmd+[` | `Cmd+⌥8` | **not** `Cmd+[` — broken |
| `Ctrl+[` | `Ctrl+⌥8` | **not** Esc — broken |

Lost: `Cmd+[` / `Cmd+]` (back/forward, indent/outdent), `Ctrl+[` (Esc in vim and readline),
`Cmd+Shift+[`/`]` (tab switching). On a Norwegian layout the Option-hidden punctuation
(`[ ] { } \ | @ $`) is exactly the shortcut-critical programming set.

**Karabiner does not rescue this** — it remaps HID events *before* the layout resolves them,
so it hits the identical wall. The wall is the layout, not where the remap happens.

### 4.4 Norwegian and Polish collide on the same mnemonic keys

|  | wants `a` | wants `o` | wants `e` |
|---|---|---|---|
| Polish | ą | ó | ę |
| Norwegian | å | ø | æ |

On a single AltGr plane one language must lose its natural positions. **This is the real
argument for two firmware layers** — stronger than the muscle-memory argument. Apple reached
the same conclusion: stock macOS U.S. puts `å` on `⌥a` and `ø` on `⌥o`.

---

## 5. Decisions taken

1. **Host layout carries the characters; firmware sends `RALT()` composites.** The only
   approach that behaves identically in every application.
2. **One custom US-based host layout per OS. Never switch input sources again.** The
   MacBook's built-in keyboard is the one whose legends lie (see §7 for the mitigation).
3. **Drop dead keys entirely.** All twelve characters live on the AltGr/Option plane.
   Removes the `'`/`"` annoyance in code, and is required anyway since `AltGr+'` is used.
4. **Host AltGr plane:**
   - Polish nine at **stock Polish (Programmers)** positions:
     `a→ą  c→ć  e→ę  l→ł  n→ń  o→ó  s→ś  x→ź  z→ż`
   - Norwegian three on the leftovers, mirroring physical Norwegian geography
     (å right of P, ø right of L, æ right of ø):
     `p→å  ;→ø  '→æ`
   - Shift+AltGr gives capitals throughout; `Cap` value `5` for Caps Lock.
5. **Two firmware language layers**, because the layer decouples finger position from host
   encoding — the ugly indirection stays invisible:

   ```
   POL layer:  Key_A → RALT(Key_A)          host: AltGr+a → ą
   NOR layer:  Key_A → RALT(Key_P)          host: AltGr+p → å
               Key_O → RALT(Key_Semicolon)  host: AltGr+; → ø
               Key_E → RALT(Key_Quote)      host: AltGr+' → æ
   ```

   Fingers see `a→å` and `a→ą`. Perfect mnemonics *and* a standards-compatible host layout.
6. **Keep stock Polish-Programmers positions on the host even though standard-keyboard
   muscle memory is not valued** — it now costs nothing (never seen) and buys a free
   fallback: on any machine where nothing can be installed, select stock *Polish
   (Programmers)* (native on Windows, Linux and macOS) and the whole Polish layer works
   unmodified. Only Norwegian degrades.
7. **Access cost mirrors the priority order:**
   - Norwegian: **held** layer shift (`ShiftToLayer`) — one motion, zero state, nothing to exit.
   - Polish: **one-shot** layer (`OSL`) — tap, tap; double-tap sticks for a run of prose,
     third tap releases. Also sidesteps the same-hand problem (Polish letters skew left-hand).
8. **Layer keys cost zero base keys** — use Qukeys on the two middle-column keys:

   ```c++
   Qukey(0, KeyAddr(2, 5), ShiftToLayer(NOR)),  // ` taps, Norwegian holds
   Qukey(0, KeyAddr(2, 6), ShiftToLayer(POL)),  // \ taps, Polish holds
   ```

   Both keep their tap behaviour; both are index-finger reachable.

---

## 6. Rejected alternatives, and why

| Alternative | Why rejected |
|---|---|
| **Norwegian host + firmware compensating to US** | Breaks `Cmd+[`, `Cmd+]`, `Ctrl+[` and tab-switching (§4.3). macOS-specific artifact; the `.klc` work becomes dead. |
| **Firmware-only via `Kaleidoscope-Unicode`** | Still needs per-OS setup and a mode switch; slow; fails in many apps and over RDP; may not fit in remaining AVR flash. Fallback only. |
| **Two stock layouts + OS-level switching** | This is the status quo and the thing being eliminated. |
| **Single AltGr plane, no language layers** | Forces `å`/`ø` onto unnatural keys (`p`/`;`) that the fingers would have to learn (§4.4). |
| **`å` on `AltGr+[`** | `[` is Fun-layer only on the Atreus — would be a three-key chord. |
| **Rename R3C6 to `Key_RightAlt`** | On Windows, an AltGr-declaring layout consumes RightAlt as Ctrl+Alt, killing Alt-Tab and Alt-F4 from that key. Both Alts are needed. |
| **Locked toggle layer (current firmware behaviour)** | Optimises for sustained accented runs that never occur; forces an explicit exit press. The originally-reported clunkiness. |

### Known cost of the chosen design

**Wrong-layer errors are a new failure mode.** `å` and `ą` become the same finger motion,
distinguished only by which layer key was taken. Typing `ą` inside a Norwegian word will
happen. Judged worth paying, but it is not free.

**Shift + one-shot layer + `RALT()` ordering** must resolve to `Shift+AltGr+a → Ą`. The
`.klc` already defines shiftstate 7 correctly, but this needs testing on hardware, not
reasoning.

---

## 7. Open item: the built-in MacBook keyboard

Used on the couch and on the train — **regular use, not rare**. So the "let its legends lie"
decision has a real cost and needs mitigation. Two options:

- **Do nothing.** Letters and digits stay correct; ~12 punctuation keys drift; `æøå` move to
  Option positions. Acceptable if couch use is mostly reading and light typing.
- **Karabiner-Elements, per-device rules** (`device_if` on the built-in keyboard's
  vendor/product ID) rewriting its `ø`/`æ`/`å` keys to emit the right AltGr combos, making
  its Nordic legends true again. This direction *works* — unlike §4.3 — precisely because
  those three keys are letters that never combine with Cmd, so there is no composition to
  break. Cost: a driver-level dependency; macOS-only (Linux has it natively, Windows does not).

Design implication either way: **the Option plane should be chosen so that the built-in
keyboard remains tolerable even with no Karabiner installed.** To be worked through.

---

## 8. Deliverables when work starts

1. **macOS** — a `.keylayout` bundle into `~/Library/Keyboard Layouts/` (built with Ukelele).
   No admin, no reboot. Best install story of the three.
2. **Windows** — extend `archive/Atreus-layout.klc`: add the nine Polish letters, remove the
   duplicate `æøå` definitions, relocate to `p`/`;`/`'`, strip the dead keys, rebuild in
   MSKLC. Roughly 60% done already.
3. **Linux** — an XKB symbols file on `us(basic)` + `level3(ralt_switch)`.
   X11: `~/.xkb/symbols/…` with `setxkbmap -I~/.xkb`, no root. **Wayland is the worst
   friction point** — GNOME/KDE ignore `~/.xkb`, so it means `/usr/share/X11/xkb/symbols/`
   plus an `evdev.xml` entry, with root, undone by package updates.
4. **Firmware** — Kaleidoscope sketch with the `NOR` and `POL` layers and the two qukeys.
   Keep in version control.

### Prerequisites / questions still outstanding

- [ ] Locate the current Kaleidoscope sketch actually flashed to the board. Needed to see
      which base keys are already freed, which plugins are carried, and how close the build
      is to the 28 KB flash ceiling.
- [ ] Confirm whether EEPROM holds a Chrysalis keymap that would override a new flash (§3).
- [ ] Extract the exact macOS `Norwegian` (ID 12) mapping table with Ukelele — the
      Option-plane positions were reasoned about, not verified byte-for-byte.
- [ ] Decide on Karabiner for the built-in keyboard (§7).
- [ ] Confirm whether a no-install fallback is needed for machines where no layout can be
      installed (client laptops, VDI, RDP). If yes, budget AVR flash for it — likely by
      dropping `MouseKeys` or `DynamicMacros`.
- [ ] Linux: X11 or Wayland on the machines that matter?

---

## 9. References

- [Stock `Atreus.ino`](https://github.com/keyboardio/Kaleidoscope/blob/master/examples/Devices/Keyboardio/Atreus/Atreus.ino)
- [`Atreus2.h` — matrix and `PER_KEY_DATA`](https://github.com/keyboardio/Kaleidoscope/blob/master/plugins/Kaleidoscope-Hardware-Keyboardio-Atreus/src/kaleidoscope/device/keyboardio/Atreus2.h)
- [`key_defs.h` — `RALT()` at line 303, `Key_RightAlt` at 247](https://github.com/keyboardio/Kaleidoscope/blob/master/src/kaleidoscope/key_defs.h)
- [Kaleidoscope-OneShot README](https://github.com/keyboardio/Kaleidoscope/blob/master/plugins/Kaleidoscope-OneShot/README.md)
- [Kaleidoscope-Qukeys README](https://github.com/keyboardio/Kaleidoscope/blob/master/plugins/Kaleidoscope-Qukeys/README.md)
- [The Atreus Layout (Keyboardio)](https://shop.keyboard.io/pages/the-atreus-layout)
- Ukelele (macOS `.keylayout` editor) — <https://software.sil.org/ukelele/>
- Karabiner-Elements — <https://karabiner-elements.pqrs.org/>
