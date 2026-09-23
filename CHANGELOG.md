# Changelog

## 0.2.0

- **Two instructions in one breath.** “A camera n 6 and B camera n 12”
  is two instructions, and each camera gets what was said to it. It used
  to collect every camera named into one list and apply the first
  instruction to all of them: both cameras got N6, and the app reported
  success — a wrong write nobody was told about. Naming two cameras for
  one change still means what it reads (“A camera and B camera 5600
  kelvin”), because what tells the two apart is whether anything was
  said between the two names. If any part of the sentence is refused,
  none of it is sent.
- **“A camera and B camera 5600 kelvin” works.** It was refused as
  “no value heard”: “and” is a number word, for “one hundred and
  eighty”, and a stray one made a junk number that failed the whole
  phrase.
- **A walkthrough, on two cameras that are not there.** Nineteen pages
  that dim the window and point at the control each one is about, run
  against two practice cameras living inside the Mac. They are not on the
  network, and for the length of it the real ones are neither written nor
  read — so a page can ask you to hold the key and say “A camera 5600
  kelvin”, and the card moves with nothing at stake. Every phrase it
  asks for is the real grammar, through the real recogniser. It opens
  once, on the first launch, and stays in the menu; skipping, finishing
  or simply closing the console all end it and put the job's cameras
  back.
- **No cameras until you add them.** A new copy used to invent A, B and C
  and show all three reporting “not answering”, which is the app making
  up a rig nobody told it about. The console now says there are none and
  offers the button, and a command with nowhere to go is refused with
  what to do about it rather than failing on the wire.
- **“What it can be told” is now Camera Controls**, which is what it is.
- **Teaching it your words** was two unexplained boxes, the second of
  which had to be a phrase you already knew by heart. It is two numbered
  steps now: the phrase you say, and the one it already knows — with a
  list to pick that one from, and a line under it saying what it actually
  does, checked as you type by the same rule that decides whether the
  button works. Presets moved to their own tab, since they were never the
  same idea.
- **Open AIcam when this Mac logs in.** A cart Mac gets power-cycled, and
  a menu bar app nobody remembers to launch is a hotkey that does nothing
  at the worst moment. The switch is in Settings and the login item is
  the operator's: macOS lists it in General › Login Items, and turning
  it off there turns the switch off here.
- **Buy Me a Beer**, in the menu and in Settings. A QR code, deliberately
  not a link — the app does not send anybody to a payment page.
- **A settings file written by an older copy survives an update.** It was
  being thrown away whole, cameras and all, if it was missing any key
  that had been added since.

### What is still not right

Both carried over from 0.1.0, both waiting on a real camera rather than a
guess.

- **“Show me log” and “show me 709” write the wrong variable.**
  `SDI1Processing` is the kind of feed a connector carries; the
  log-versus-look choice is `SDI1Gamma` — and it is spelled differently
  on every camera generation. Everything else on the SDI side works; this
  one needs a real camera to confirm how the values are encoded before it
  is changed. See `docs/processing.md`.
- **A sequence is not confirmed.** A preset sends its writes one after
  another and moves on when the camera answers, and that answer means
  accepted rather than applied. See `docs/sequences.md`.

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
- **Camera Controls**: every setting the app may write, with the
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
