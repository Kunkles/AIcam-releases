# Changelog

## 0.7.0

**Nothing you can see has changed.** Every command works exactly as it did
in 0.6.0, because what this release adds is not wired to anything yet. It
is here so it ships, is versioned, and can be pointed at a camera.

### A client for ARRI's Camera Access Protocol

CAP is the documented way into the same cameras the Web Remote CGI
reaches, and it is better in the ways that have cost this project the
most:

| | Web Remote CGI | CAP |
| --- | --- | --- |
| Status | undocumented, inferred from the browser remote's JavaScript | specified, versioned |
| Updates | poll and diff it yourself | the camera pushes |
| Values | untyped, meaning guessed | typed, and the type is part of the variable |
| Camera letter | inferred from the IP address | the camera states it |
| Clients | as many as you like | 4, or 1 on an SXT/LF/65 |

Two of those are bugs this release makes impossible rather than fixes.
The exposure index was wrong on every model in 0.5.0 because the CGI
takes it as a position in a list that differs per camera; over CAP it is
a number carrying the number, beside the camera's own list of them. And
the A/B/C letter is a variable, where the CGI path infers it from the
address — a convention, not a fact.

In both engines, `aicam/cap/` and `mac/Sources/AIcamCore/CAP*.swift`.

### How far it is trusted

The framing is checked against the specification's own example dumps,
byte for byte, in both languages — the challenge reply, the password
command, a subscribe, and a media-status array that decodes to exactly
the tuple ARRI says it represents. The authentication example reproduces
their published digest only when the password is `arri`, which is how the
scheme is known to be MD5 of the password *followed by* the challenge.

The Swift connection tests start the Python fake camera and drive the
Swift client against it. Two implementations agreeing with each other is
worth more than either agreeing with itself, and it is the only check
that would catch them making the same mistake in different words.

### What still needs a camera

What an unset password does. The ND density scale, which the
specification leaves to the camera's own list. Whether the beacon
interval matters in practice. And which variables a given body and SUP
actually have.

## 0.6.0

The theme of this one is values the app decided instead of asked for.
Three of them were wrong, one of them on every camera, and the reason
none had been caught is the same reason "show me log" survived three
releases: the fake cameras held the same beliefs the app did.

### The exposure index was wrong on every model

The camera takes an EI as a position in its own list. AIcam carried one
hardcoded list of sixteen and sent a position in that. The lists are not
the same — an AMIRA and an ALEXA Mini have no EI 1000, a Mini LF starts
at 200 rather than 160, an ALEXA 35 has four low-noise entries after
4800 — so:

| | wrong |
| --- | --- |
| Mini LF | **16 of 16** — say 800, get 1000 |
| AMIRA, ALEXA Mini | 8 of 16 — say 1000, get 1280 |
| ALEXA 35 | 1 of 16 — 6400 reached into the low-noise entries |

A third of a stop to a stop out, accepted by the camera, reported as
done. Enhanced sensitivity was out by one on the 35 for the same reason.

The list is read off the camera now and the spoken number matched against
it. A camera that has not got that EI refuses and lists what it has. A
camera that publishes no list falls back to the shipped table and says in
the result that it did, because the shipped table is the thing that was
wrong.

### A preset is not sent unless the camera has all of it

Presets are saved from one body and spoken at another, and the models do
not carry the same values: a 2x anamorphic desqueeze is `1_33`, which an
ALEXA 35, an AMIRA and an ALEXA Mini have and a Mini LF has not. It used
to be posted anyway and left to the camera to refuse — but an ARRI camera
answers `{"result": 1}` for ACCEPTED, not applied, so a preset made for
another body could report as done having changed nothing.

Now the camera is asked first, and a preset it cannot take is refused
whole, naming the setting: *"this camera has not got LensSqueezeFactor
1_33 — nothing was sent"*. Same rule as two instructions in one breath.

ND and the recorded-image flip were checked for the same fault and are
clean: they are written by name rather than by position, and every value
the language writes exists on all four models.

### Looks and framelines can be asked for by name

- **A look can be loaded by name.** "A camera load the commercial look",
  "A camera look vibrant", "A camera look lcc 709". The camera is asked
  what it has and the words are matched against the installed list. A
  look is written by its POSITION, though, so a wrong one is accepted
  rather than refused — the write is read back, and what gets reported is
  the name the camera says is loaded afterwards.
- **"The look" still means the look.** "Show me the look" and "go back to
  the look" are unchanged. The rule is what is left in the sentence:
  nothing, and it is that command; a word, and the word names a file.
  Vosk writes "to" as "two", so the leftover is judged in the same
  spelling the intent was matched in, or "go back two the look" would
  have loaded a look called two.
- **A frameline can be asked for by name.** A file holding several
  rectangles — a 16:9, a 9:16 and a 1:1 for one delivery — has no single
  ratio, and a ratio was the only way to ask. Say a distinctive word from
  the file name instead: "A camera frameline spherical". A word that
  matches nothing is dropped rather than refused, and two files answering
  equally well is a refusal listing both — never a silent pick.
- **A file name holding several ratios no longer invents one.**
  `Spherical_16-9_9-16_1-1.xml` had the "9" and the "9" out of the middle
  of it read as 9.90 and was offered in refusals as a ratio that camera
  had. A name carries a ratio only when it carries exactly one.
- **Framelines and looks share one matcher.** The row is scanned for the
  file name rather than indexed into, which turned out to be necessary
  rather than careful: there are THREE row shapes across the four
  cameras, and the 35 calls its looks `.ALF4` rather than `.aml`.

### The test doubles can disagree with the app now

- **The fake cameras no longer import the app's EI table.** It opened
  `EI_TABLE = SELECTABLE_EI + ES_EI` — the stand-in and the thing it
  stood in for holding one belief between them. Each fake now publishes
  its own model's list, from the catalogue.
- **The fixture generator writes both copies.** The Swift tests read the
  shared fixture from their own bundle and it was kept in step by hand,
  so a fixture written to one copy left the Swift side held to the last
  one — exactly the drift the fixture exists to catch.

### Also

- **A voice coverage sheet**, generated by `tools/reference_sheet.py`
  from the language file and the model catalogues: every menu item that
  can be spoken to, the ten reachable only inside a saved preset, and
  seven menu areas on all four cameras that nothing can be said to yet.

### What is still not right

- **Accepted is not applied.** `{"result": 1}` means the camera took the
  write, not that it did anything. Only the look path reads back. Whether
  a read-back is reliable enough to do everywhere needs a camera.
- **The AMIRA's frameline gate.** `SDICenterMark` has no OFF, so "no
  centre mark" there has no value to write.
- **Discovery and the frameline tables are still unconfirmed.** ARRI's
  simulators mark the file tables as local, so nothing has yet proved a
  real camera publishes them over the CGI.

## 0.5.0

- **“Show me log” and “show me 709” work.** They wrote
  `SDI1Processing`, which is the kind of feed a connector carries, not
  the log-versus-look choice — so every ALEXA would have refused them.
  The right variable depends on the camera, and there is no way to know
  it without asking, so the camera is now asked: `/all.cgi` is read
  before the write, and what goes out is the first name that camera has
  carrying the first value that name takes. An AMIRA gets
  `SDI1PathProcessing`, a Mini LF `SDI1Gamma`, an ALEXA 35
  `SDI1GammaPIA` with LogC4, and the 35's finder
  `EVFMonitorPathProcessingPia`, which has no plain LOOK.
- **709 is two writes**, because it is not a processing mode on any
  ALEXA: the look, and the colour space to REC709.
- **The centre mark, the level and the outside shading were wrong on the
  older cameras** for the same reason — an AMIRA spells it
  `SDICenterMark`, not `SDI1CenterMark` — and are fixed by the same
  change.
- **A camera that cannot do something says so.** An ALEXA Mini has no
  centre mark on its SDI outputs under any spelling; “centre dot” there
  now answers “this camera has nothing that does that” rather than
  failing on the wire and looking like a network fault.

### What is still not right

Most of what this list held is now fixed. What is left needs a camera,
not a decision.

- **Whether the CGI takes an enum by name or by index.** Everything
  writes the option name, which is what `/all.cgi` publishes. The reason
  to doubt it is `ExposureIndex`: the simulator lists `EI_160`, and a
  real camera takes an integer index instead. If these are indices too,
  the shape of the fix is unchanged and the values move.
  `tools/check_camera.py --write` settles it in one run.
- **Which of two variables an ALEXA 35 honours.** It has both
  `SDI1Gamma` and `SDI1GammaPIA`. This writes the Pia one, because LogC4
  is what a 35 records; only a camera can say whether that is the one
  that moves the picture.
- **The AMIRA gates its centre mark behind framelines being on**, and
  its `SDICenterMark` publishes no `OFF`. So “no centre mark” there has
  no value to write and needs another mechanism — which no amount of
  choosing the right variable name fixes. See `docs/surfaces.md`.
- **A sequence is not confirmed.** A preset sends its writes one after
  another and moves on when the camera answers, and that answer means
  accepted rather than applied. See `docs/sequences.md`.
- **Discovery has never met a real camera.** The 1.2 second timeout is
  taste rather than measurement. See `docs/discovery.md`.

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
