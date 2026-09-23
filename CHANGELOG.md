# Changelog

## 0.4.0

- **A drawn menu bar icon** — a camera with a waveform through it — in
  place of the system waveform glyph. It is a template image, which means
  the artwork is black and macOS paints it: light on a dark menu bar,
  dark on a light one, inverted while the menu is open. Listening is the
  same shape knocked out of a filled slab rather than a tint, because a
  tint is what the system already uses for everything else and this has
  to be readable from across a cart.
- **A phrase is no longer “already taken” because a word was thrown
  away.** Teaching “a camera go 125 anamorphic” for a 1.25× squeeze was
  refused as already meaning shutter 125 — which it only does because the
  grammar discards words it does not know, so the anamorphic went on the
  floor and the 125 read as an angle. A phrase carrying a word it does
  not understand is free to be given one. A phrase it understands
  completely is still protected.
- **A taught phrase that writes something the camera will not take says
  so**, under the phrase, in red, with the nearest value the camera does
  offer as a button. Pressing it repoints the phrase; the wording was
  never the problem. Nothing can be taught a value the setting does not
  list any more either. This is not hypothetical: four phrases on the
  machine this was built on wrote squeeze factors of 1_3 and 1_5, which
  no ALEXA has ever offered, because they were taught against a stand-in
  whose list was wrong.
- **For anyone with a camera to hand**: `tools/check_camera.py` reads
  `/all.cgi`, keeps the dump, and holds what ARRI's simulators say
  against what the camera actually publishes — every variable this app
  may write, whether it is there, and what it takes. With `--write` it
  tries each one, reads it back, times it and puts it back. See
  `docs/on-a-camera.md`.

### What is still not right

The shape of this got clearer, and worse, since 0.3.0. The variable names
in the language are an ALEXA 35's, and about half the overlays are wrong
on at least one other camera — the AMIRA and the ALEXA Mini do not
number their SDI settings at all. `docs/surfaces.md` has the table;
`tests/test_against_the_models.py` holds six failing cases as the score.

- **“Show me log” and “show me 709” write the wrong variable**, on every
  model. `SDI1Processing` is the kind of feed a connector carries; the
  log-versus-look choice is `SDI1PathProcessing`, `SDI1Gamma` or
  `SDI1GammaPIA` depending on the camera. 709 is not a processing mode at
  all — it lives in `SDI1ColorSpace`. See `docs/processing.md`.
- **The viewfinder is wrong on an ALEXA 35 too.** 0.3.0 said the finder
  side was fine; that is true of the AMIRA, the Mini and the Mini LF. On
  a 35 the live variable is `EVFMonitorPathProcessingPia`, and it has no
  plain `LOOK`.
- **The centre mark, the level and the outside shading are wrong on the
  older cameras**, which spell them `SDICenterMark` rather than
  `SDI1CenterMark`. An ALEXA Mini appears to have no SDI centre mark at
  all, and the AMIRA gates its centre mark behind framelines being on,
  which no amount of choosing the right variable name fixes.
- **A sequence is not confirmed.** A preset sends its writes one after
  another and moves on when the camera answers, and that answer means
  accepted rather than applied. See `docs/sequences.md`.
- **Discovery has never met a real camera.** The 1.2 second timeout is
  taste rather than measurement. See `docs/discovery.md`.

## 0.3.0

- **Find cameras.** Instead of typing an address, a sweep of the network
  this Mac is actually on, keeping whatever answers the way an ALEXA
  does — with its model and serial, added to the next free letter at the
  address it answered on. It does not use the app's default addresses to
  look: those are one crew's convention, and the machine is usually
  somewhere else entirely, so the networks searched are read from the
  Mac's own interfaces and named on screen. What counts as a camera is
  the shape of the answer rather than the status code, because plenty of
  things on a network return 200 and a printer in the camera list would
  be worse than an empty one. Nothing is written during a sweep; it reads
  `/all.cgi` and nothing else.
- **A sweep macOS has not allowed says so.** The first time, macOS asks
  whether AIcam may find devices on the local network — and until it is
  allowed, every request fails instantly and the result looks exactly
  like a network with no cameras on it. The difference is the clock: 254
  addresses take about seven seconds when the requests really go out and
  under two when they do not. A result that came back too fast points at
  System Settings, and says to quit from the menu bar icon rather than
  just closing the window, because the permission only reaches a new
  process.

  **macOS asks again after every update**, this one included. If Find
  cameras comes back instantly having found nothing, that is what
  happened.

### What is still not right

- **Discovery has never met a real camera.** The mechanism is tested —
  against stand-ins answering over HTTP, and against a real router, which
  is correctly not listed — but no ALEXA has answered a sweep. The
  timeout of 1.2 seconds is taste rather than measurement, and a camera
  slower than that is missed silently, which looks identical to an empty
  network. `docs/discovery.md` lists what a camera has to settle.
- **“Show me log” and “show me 709” write the wrong variable.**
  `SDI1Processing` is the kind of feed a connector carries; the
  log-versus-look choice is `SDI1Gamma` — and it is spelled differently
  on every camera generation. Everything else on the SDI side works; this
  one needs a real camera to confirm how the values are encoded before it
  is changed. See `docs/processing.md`.
- **A sequence is not confirmed.** A preset sends its writes one after
  another and moves on when the camera answers, and that answer means
  accepted rather than applied. See `docs/sequences.md`.

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
