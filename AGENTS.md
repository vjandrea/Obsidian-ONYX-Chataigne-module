# Obsidian ONYX Chataigne module: design notes

Custom Chataigne module (`type: "OSC"`) to control Obsidian ONYX over its native OSC implementation.

## Source material

- Manual page: https://support.obsidiancontrol.com/Content/Onyx_Manual/Networking/OSC.htm (how to enable OSC on the console, network setup)
- Full address list: `references/OSC-Mapping-v1.20.pdf` (vendored, fetched from `https://files.obsidiancontrol.com/s/yndNn7FsaB9Fyay`). This PDF is the only source of truth for addresses below: it's revision 5 of v1.20, 29 pages, and we've read it in full.
- Reference implementation pattern: `~/dev/Madrix5-Chataigne-Module` (Andrea's own Chataigne module for Madrix 5, same `type: "OSC"` shape). Its `module.json` has no `"values"` or `"parameters"` keys at all: it relies purely on `"commands"` + `autoAdd: true`, and every command callback calls `local.send(address, ...args)` with an address built as a plain JS string. We're following the same pattern here.
- Chataigne engine source (for how `type: "OSC"` custom modules actually work): `~/dev/Chataigne/Source/Module/modules/osc/custom/CustomOSCModule.cpp` and `Source/Module/Module.cpp` (`setupModuleFromJSONData`, `createControllablesForContainer`). `Source/Module/Module.cpp:230` shows `commands` entries become `ScriptCommand`s that just call the named JS function; there's no built-in per-command "address" templating, all address construction happens in the script.

## Obsidian ONYX OSC essentials

- Every address is prefixed `/Mx/...` where `x` is the **Device Space** number configured on the console (Network > OSC). This is a console-side setting, not something OSC itself negotiates. Our module needs one plain config parameter (e.g. `Device Space`, Integer, default `1`) that the script reads to build the `/M<n>/` prefix; it's local module config, not itself mapped to an OSC address (except see "Configuration" below, which is a distinct remote nudge of that same console setting).
- Two address flavors per control, not always both present:
  - **Update Address**: ONYX → us, feedback (led state, color, text, fader position). Types: `int`, `float`, `string`, `color`, sometimes `.../led`, `.../led/color`, `.../led/blink`, `.../text`, `.../text/color`.
  - **Execute Address**: us → ONYX, triggers the action. Type `up/down` (button press/release as 0/1) or `float` (faders) or `string` (rare).
- **Feedback handling: don't declare `values` in `module.json`.** Turn on `autoAdd: true` (like Madrix5) so Chataigne creates value objects on the fly for whatever Update Address the console actually sends. Declaring ~150 individual feedback values by hand would be pure bloat for something the module framework already does generically.
- **Commands: generic/scripted, not one JSON entry per address.** The spec is ~300 addresses; most are members of small regular families (playback bank number, view number, channel index...). Per user decision, model each family as one parametrized `command` (bank/index/action as command parameters) that computes the actual address string in `moduleScript.js`, the same way the manual itself documents the Playback Pages syntax generically (`/Mx/playback/page<pageIndex>/<buttonIndex>/<action>`). Only the handful of genuinely singular controls (transport buttons, master faders, keypad, command line) get their own one-off named command, mirroring how Madrix5's module.json has one command per control rather than per numeric family.

## Address catalog (by manual section)

`Mx` = `/M` + Device Space number, omitted below for brevity (e.g. "button/4201" means "`/Mx/button/4201`").

### Commandline (p.1)
- `commandLine/0001` (status: color, text, text/color) - feedback only
- `commandLine/0002` (command: color, text, text/color) - feedback only (no execute address given for typing a command)

### Playback (p.1-13) - license required
20 playback banks, each with buttons A/B/fader/C/D:
```
base(bank) = bank <= 10 ? 4200 + (bank-1)*10 : 4600 + (bank-11)*10
A     = button/(base+1)       up/down, led, led/color, led/blink
B     = button/(base+2)       up/down, led, led/color, led/blink
fader = fader/(base+3)        float 0-255, color
C     = button/(base+4)       up/down, text (<name>), color, text/color
D     = button/(base+5)       up/down, led, led/color
```
Also fixed (not per-bank):
- `button/4600` Fader Swap (up/down)
- `button/4412` / `button/4413` Bank Page Up / Down (up/down, + led feedback)
- `scroll/4110` /up /down Bank Scroll (up/down)
- `scroll/4411` /up /down Bank Page Scroll (up/down)
- `button/4421..4425` PLAYBACK BANK 1-5 quick-select tabs (text, color, text/color, up/down execute)
- `label/4401` PLAYBACK BANK NUMBER (text, color, text/color) - feedback only
- `button/5502` SELECT, `button/5503` RELEASE, `button/5504` BEAT, `button/5511` SNAP, `button/5512` `||`/Back, `button/5513` GO (all: up/down, led, led/color, led/blink)
- `button/4121` View (up/down, led)

### Playback Pages (p.12-13) - license required, generic syntax, no update/feedback messages exist for these
```
/Mx/playback/page<pageIndex>/<buttonIndex>/<action>   type up/down
pageIndex: 1-100 (one-based), buttonIndex: 0-99 (zero-based)
action (not case sensitive): go|play, pause, release, select, snapgo, toggle, back
```
This is the one family the manual itself documents generically, model the command the same way: parameters `pageIndex`, `buttonIndex`, `action`.

### Master Faders (p.13) - fixed set of 4, no formula needed
```
button/2201 GRAND MASTER FLASH (led+execute) -> fader/2202 GRAND MASTER LEVEL (float 0-255)
button/2211 FLASH MASTER FLASH               -> fader/2212 FLASH MASTER LEVEL
button/2221 GROUP MASTER A FLASH             -> fader/2222 GROUP MASTER A LEVEL
button/2231 GROUP MASTER B FLASH             -> fader/2232 GROUP MASTER B LEVEL
```

### Screens / Views (p.14-16)
16 view buttons, two contiguous ranges:
```
view(n) = n <= 8 ? button/(1100+n) : button/(3100+(n-8))
```
(`up/down`, `led`, `led/color`, `led/blink` for each)

### Keypad (p.16-20)
Mostly one-off named buttons, fixed addresses (not a numeric family worth a formula):
`button/2001` MACRO, `2002` PREVIEW, `2003` MENU, `4321` FADE, `4322` DELAY, `4331` SNAPSHOT, `4332` BANK, `5101` EDIT, `5102` UNDO, `5103` CLEAR, `5104` COPY, `5106` MOVE, `5107` DELETE, `5401` RECORD, `5402` UPDATE, `5411` LOAD, `5412` GROUP, `5413` CUE.
Numeric pad is a clean sequential family:
```
digit(0-9) = button/(5200+digit)   e.g. digit 0 -> button/5200
button/5210 "-", 5211 "+", 5212 ".", 5213 Enter, 5214 "/", 5215 Backspace, 5216 "@"
button/5301 Full, 5302 Through
```
All: `up/down` execute only (no feedback listed for these besides the small set above with led entries).

### Programmable Function-Keys (p.20-22)
12 keys, **irregular** address scheme, don't try to formulaize, use a lookup table:
```
F1  button/5601   F2  button/56A1   F3  button/5602   F4  button/56A2
F5  button/5603   F6  button/56A3   F7  button/2101   F8  button/21A1
F9  button/2102   F10 button/21A2  F11 button/2103   F12 button/21A3
```
(`up/down`, `led`, `led/color`, `led/blink` each)

PF Groups: 5 fixed slots + scroll
```
button/5701..5705  PF GROUP 1-5 (text, color, text/color, up/down execute)
scroll/5706 /up /down  PF GROUP Scroll
```

### Programmer (p.23-24)
One-off named buttons: `button/6401` Last, `6402` Next, `6411` Swap Programmer, `6001` HIGHLIGHT, `6003` CV, `6108` Link (execute only, no update).

Base/Effect Channel Groups: 5 fixed slots each
```
button/6101..6105  BASE CHANNEL GROUP 1-5     (text, color, text/color, up/down)
button/6201..6205  EFFECT CHANNEL GROUP 1-5   (text, color, text/color, up/down)
```
Base/Effect Channels within the active group: 4 channels each, regular step of 10
```
baseChannel(i)   = button/(6100 + i*10 + 1)   i = 1..4   (led, led/color, led/blink, up/down)
effectChannel(i) = button/(6200 + i*10 + 1)   i = 1..4
```
Value belts (feedback-heavy, channel name/value display), same index step of 10:
```
belt/(6100+i*10+2)   VALUE BASE i    (channelName, channelValue, color, channelValue/color, channelName/color; up/down execute on the bare address)
belt/(6200+i*10+2)   VALUE EFFECT i
```

### Track Function / Navigation (p.28-29)
`button/7001` TRACKFUNC P/T, `button/7004` MODE, `button/7301..7304` NAVIGATE UP/LEFT/DOWN/RIGHT (all `up/down`, `led`, `led/color`, `led/blink`).

### Configuration (p.29)
```
configuration/deviceSpace           string, feedback of current Device Space ID
configuration/deviceSpace/up        up/down, nudges Device Space ID up
configuration/deviceSpace/down      up/down, nudges Device Space ID down
```
Note this is a *remote control of the console's own Device Space setting*, distinct from our module's local `Device Space` parameter used to build the `/Mx/` prefix for every other address. Don't conflate the two.

## Licensing / OSC trial gotcha (troubleshooting)

ONYX gates OSC (and MIDI, Timecode) by license/hardware mode, this is the first thing to check when "OSC stopped working" comes up:

- **FREE** (no Obsidian USB device or NETRON node attached) and **NOVA** (NX-DMX device or NETRON node attached) modes: OSC/MIDI/Timecode are **disabled by default**, but can be started as a **5-minute trial** (click the FREE/NOVA license indicator bottom-left, or the button bottom-right of the OSC config page). Once the trial window closes, ONYX must be **fully restarted** to get another 5-minute trial, it does not renew on its own.
- **NOVA+** (NX-Touch/NX-K/NX-P attached) and **LIVE 8/16/32/64/128** (8/16/32 Key, Premier Key/NX Wing/NX 2, Elite Key attached): OSC/MIDI/Timecode are **fully enabled**, no trial limit.
- Practical troubleshooting order when a Chataigne↔ONYX link that used to work goes silent: (1) is real Obsidian hardware (or a NETRON node) actually connected and recognized, not just "FREE" showing bottom-left, (2) if running FREE/NOVA on purpose, has the 5-minute trial simply expired, restart ONYX and re-arm the trial, (3) only then look at network/port/device-space settings.
- Sources: https://support.obsidiancontrol.com/Content/License_Information/OSC_MIDI_Timecode_Trial.htm , https://support.obsidiancontrol.com/Content/License_Information/Onyx_PC_Modes.htm

## Chataigne module API drift since this repo's scaffold

This repo's `module.json`/`moduleScript.js` are the ~2018-era `tommag/Sample-Chataigne-module` scaffold. Checked the actual engine source (`~/dev/Chataigne`, currently 1.10.4b6, full git history back to 2016) plus the sibling `~/dev/Madrix5-Chataigne-Module` (Andrea's own module, built against current Chataigne and confirmed working) to see what's stale. Verdict: mostly additive, one real breaking bit.

**Breaking: OSC I/O config block.** The scaffold's `"defaults": {"oscInput": {"localPort": 9001}}` no longer matches anything. Current `OSCModule` (`Source/Module/modules/osc/OSCModule.h/.cpp`) exposes:
- `"OSC Input"` (an `EnablingControllableContainer`): `enabled` (bool) + `Local Port` (shortName `localPort`).
- `"OSC Outputs"` (a `BaseManager<OSCOutput>`, i.e. a *named list*, multiple simultaneous outputs): each entry has `local` (bool, "use local/broadcast"), `remoteHost`, `remotePort`, `listenToOutputFeedback`.
Madrix5's module.json confirms the current shape:
```json
"defaults": {
  "autoAdd": true,
  "OSC Outputs": { "OSC Output": { "local": true, "remotePort": 9001 } },
  "OSC Input": { "enabled": false, "localPort": 9002 }
}
```
Use this shape, not the scaffold's `oscInput`.

**Additive, worth using:**
- `CustomOSCModule` (`Source/Module/modules/osc/custom/CustomOSCModule.cpp`) gained params the 2018 scaffold never had: `Split Arguments`, `Use Hierarchy`, `Auto Feedback`, `Color Send Mode` (enum: Color/RGB Float/RGBA Float), `Boolean Send Mode` (enum: Int/Float/T-F), and a `Clear Values` trigger. None of these need touching for our design, defaults are fine, but `Auto Feedback` is worth knowing about (it would echo Chataigne-side value edits back out over OSC automatically, we don't want that since our values are meant to be pure inbound feedback from the console).
- As of `79e37c7c` (2025-01-02), `autoAdd`/`Split Arguments` are **no longer auto-defaulted or auto-hidden** by the engine. Our module.json must explicitly set `"defaults": {"autoAdd": true}` for feedback to work, and explicitly list `"hideDefaultParameters": ["autoAdd"]` if we want it hidden from the module's UI panel, exactly as already planned below, just confirming it's still necessary and not implicit.
- Script API on `local` (the module's own OSC scriptObject, `Source/Module/modules/osc/OSCModule.cpp:77-80`): `local.send(address, ...args)` (what `madrix5.js` uses, unchanged), plus newer (2021-2022) additions `local.sendTo(ip, port, address, ...args)`, `local.match(pattern, address)` (bool wildcard match), and `local.register(pattern, callbackName)` to get a scoped callback only for addresses matching an OSC pattern (supports wildcards).
- A global `oscEvent(address, args, senderIP)` script function, if defined, fires for **every** incoming message regardless of `autoAdd`/declared values (`OSCModule.cpp:174-201`). This is a real alternative (or complement) to leaning on `autoAdd`: we could parse ONYX's numeric `/Mx/button/NNNN` addresses ourselves in `oscEvent` and set friendly named values, rather than letting `autoAdd` create opaque numeric-named ones. Worth prototyping both and picking whichever reads better in the Chataigne UI.
- `module.json` `"commands"` (`menu`/`callback`/`parameters`) mechanism is unchanged (`ScriptCommand.cpp`, feature since 2018-10-31), our planned command-per-family approach needs no rework here. New optional `"setupCallback"` property (fires once on command creation) exists but we don't need it.
- Parameter/value JSON grew nested `"type": "Container"` grouping, `index`/`collapsed` layout hints, `description`, `customShortName`, `readOnly`, min-only/max-only support, since the scaffold was written. Purely additive, doesn't force any changes to our plan, just more knobs available if the UI needs organizing (e.g. group all Playback commands under a "Playback" menu, already possible via the existing `"menu"` command property).

## Implementation plan (not yet built)

1. `module.json`: `type: "OSC"`, `hasInput`/`hasOutput` true, `defaults`: `autoAdd: true` plus the current `"OSC Input"`/`"OSC Outputs"` shape (see drift notes above, not the scaffold's stale `oscInput` key), `hideDefaultParameters: ["autoAdd"]`, one custom Integer parameter `Device Space` (default 1), `scripts: ["onyx.js"]`, and `commands` covering:
   - Generic/parametrized: Playback Button/Fader (bank 1-20 + A/B/C/D/fader), Playback Page (pageIndex, buttonIndex, action), View (1-16), Function Key (1-16... 1-12), Base Channel / Effect Channel (group 1-5, channel 1-4).
   - One-off named: Command Line, transport (Go/Pause/Release/Select/Beat/Snap/Back), Master Faders x4, Keypad (digits + specials), Programmer buttons, PF Groups, Bank paging/scroll, Navigation, Device Space up/down.
2. `onyx.js`: a `prefix()` helper building `/M<n>` from `local.parameters["Device Space"].get()`, small formula helpers per family above (playback bank base, view base, channel base), a literal lookup array for the 12 F-keys, and one `local.send(...)` call per command callback, exactly like `madrix5.js`.
3. No manual `values` block: verify in a real Chataigne session that `autoAdd` correctly surfaces feedback for the deeper addresses (`led/color`, `text/color` suffixes) the same way it does for Madrix5's flatter address space, since ONYX's addresses are numeric IDs rather than named hierarchy segments, this is the one assumption worth testing early.
