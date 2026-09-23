# Changelog

## 0.1.0

First release of the cart tool.

- **It lives in the menu bar.** On set the app is never the thing you are
  looking at, so there is no window in the way: a hotkey a Stream Deck can
  send starts it listening from inside any application, and `aicam://listen`
  does the same for anything that can open a URL.
- **It answers on top of everything.** What it heard and what it did appear
  in the top right of the main display, over whatever is in front —
  including a full-screen app, which is where the monitoring software
  lives. A refusal stays longer than a success, because a refusal is
  something to read.
- **Voice.** White balance, tint, ASA and ES, shutter angle, ND, the
  monitoring path, the centre mark, the level, outside shading, framelines
  by ratio, and the flip and flop for Steadicam low mode — said the way a
  set says them. "5600 on a", "n twelve", "three twenty ASA", "A camera go
  to low mode".
- **The monitors and the viewfinder are separate.** "Show me log" is the
  monitors; the finder is only ever included when it is named, because it
  is the operator's own picture.
- **Undo.** Every write reads what was there first, so "undo that" puts it
  back — including a preset, and including a look that was a LUT.
- **Presets** captured from a camera somebody has already set up by hand:
  what "anamorphic" means is the job's business, not the program's.
- **Teaching.** A phrase can be taught to mean one that already works, and
  a setting nothing says yet can be given words — with the values read from
  the camera rather than typed and hoped for.
- **What it can be told**: every setting the app may write, with the
  phrases that reach it and a mark against the ones nothing says.
- **Camera models it has never met.** The settings of an ALEXA 35, a Mini
  LF, an ALEXA Mini and an AMIRA ship with the app, read from ARRI's own
  menu simulators, so the training screen works at home with nothing
  plugged in. A camera actually on the network beats them, and is
  remembered.
- **It cannot format a card, stop a take, or reboot a camera.** Thirty-one
  variables are writable, checked at the last point before the wire, under
  the grammar and under anything taught on the day.

### What is not right yet

- **"Show me log" and "show me 709" write the wrong variable.**
  `SDI1Processing` is the kind of feed a connector carries; the
  log-versus-look choice is `SDI1Gamma` — and it is spelled differently on
  every camera generation. Everything else on the SDI side works; this one
  needs a real camera to confirm how the values are encoded before it is
  changed. See `docs/processing.md`.
- **A sequence is not confirmed.** A preset sends its writes one after
  another and moves on when the camera answers, and that answer means
  accepted rather than applied. See `docs/sequences.md`.
