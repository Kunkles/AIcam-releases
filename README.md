# AIcam

Voice control for ARRI ALEXA cameras. Say it the way you would on set:

```
A camera go to 5600 kelvin
5600 on a                       # the short way, when everyone knows the context
camera B white balance thirty two hundred
C camera EI sixteen hundred
all cameras forty three hundred kelvin
ok go back to the look
```

Speech becomes a validated change and goes to the camera over its Web
Remote CGI API — the same one ARRI's browser remote uses.

Headed for a handheld: an Orange Pi Zero 3W with a 240x240 screen and
buttons, running Whisper locally so nothing leaves the set.

## The Mac app

```bash
Scripts/build.sh          # dist/AIcam.app
Scripts/release.sh        # signed, notarised, stapled
```

Everything is inside it — Python, the speech model, vosk's library,
PortAudio — so it is a drag to Applications and nothing to install. About
100MB.

Hold the button (or the space bar) and talk; let go and it goes. The
cameras are down the left with what they are holding, refreshed every two
seconds, and what was heard and what it did is down the right. It is the
same grammar as the command line: same phrases, same refusals, same undo.

Notarising needs a credential, once per Mac:

```bash
xcrun notarytool store-credentials aicam-notary --apple-id you@example.com --team-id 77F3B9355E
```

Built with Python 3.13, because PyInstaller does not do 3.14 yet, and with
`python-tk@3.13` for Tk. `Scripts/build.sh` makes that environment itself.

The hardened runtime needs three things said out loud in
`packaging/entitlements.plist`: the microphone, and — because a bundled
Python writes code in memory and loads libraries it shipped with —
unsigned executable memory and library validation off.

Cameras are set in the app — **edit**, above the camera list — and kept in
`~/Library/Application Support/AIcam/cameras.json`. The subnet and which
letter starts it are enough for most jobs; a camera somewhere else entirely
gets an address of its own, and the grey text shows what the subnet would
have given it. `--camera A=host` still works from a terminal and wins for
that run without overwriting what was saved.

## Try it without a camera

Three fake ALEXAs, and a page showing what each one would look like — the
colour of the white balance, the shutter disc at its angle, the picture the
right way up or not, and what is going out of SDI:

```bash
python3 tools/fake_cameras.py
```

```bash
open http://127.0.0.1:8800
```

```bash
python3 -m aicam --cameras A,B,C --camera A=127.0.0.1:9001 --camera B=127.0.0.1:9002 --camera C=127.0.0.1:9003
```

Say "B camera go to low mode" and the picture turns over with a low mode
badge on it; "show me log" washes the frame out. It is a stand-in for the
picture, not for a camera: nothing there proves a variable name is right on
a real ALEXA.

Kelvin's `tools/mock_camera.py` also works, and prints a line per change.

Type phrases the way you would say them. Typing a line stands in for the
microphone; pressing Return to confirm stands in for the button on the HAT.

```bash
python3 -m unittest discover -s tests -t .
```

## What each camera model has

`tools/scrape_simulator.py` reads a model's whole parameter list out of
ARRI's menu simulator — every variable, its type, its range and the values
it takes — into `data/models/`. The app ships those, so a camera model it
has never met can still be described:

```bash
python3 tools/scrape_simulator.py a35 mini-lf
```

| model | `SystemCameraType` | published settings |
| --- | --- | --- |
| ALEXA 35 | 4 | 579 |
| ALEXA Mini LF | 3 | 469 |
| ALEXA Mini | 2 | 610 |
| AMIRA | 1 | 610 |

Each simulator reports its own `SystemCameraType`, so the model numbers are
the cameras' own word rather than a guess. The ALEXA LF, the SXTs and the
older cameras have a different kind of simulator with no parameter model in
it, and are not covered.

A camera that is actually on the network beats this, and its catalogue is
kept per model in `~/.config/aicam/camera-models.json`. The simulator
mimics one SUP; a camera in the room is the SUP you have.

**What the simulator gives is structure, not always encoding.** It lists
`ExposureIndex` as `EI_160`, `EI_200`, … while a real camera takes an
integer index — which is how Kelvin has been setting EI for a year. Treat
the variable names and the shape of a setting as sound, and the exact value
a write should carry as something a camera has to confirm.

## The language is data

Everything this understands — the phrases, the settings and their ranges,
the outcomes and the variable each one writes — is in
[data/language.json](data/language.json). The Python engine reads it, the
Swift app reads it, and `tools/make_phrases.py` turns it into the sentences
the recogniser is taught to expect. Add a phrase once and all three have
it; there is a test that edits the file in a temporary copy and checks the
new phrase is understood without touching any code.

**What a phrase may write is NOT data.** That list is
[aicam/writable.py](aicam/writable.py), in code, alone in a file that
imports nothing. Anything the language names that is not on it is dropped
when the language loads, and reported. Editing data is meant to be easy —
which is exactly why it cannot reach a card or a take.

## How it is put together

| | |
| --- | --- |
| `grammar.py` | heard phrase in, a Command, Outcome, Restore or Rejection out |
| `numbers.py` | "fifty six hundred" and "5600" are the same number |
| `intents.py` | phrases that carry their own value: "show me log" |
| `commands.py` | what a valid change is, in camera-neutral terms |
| `context.py` | who you were just talking to, for a couple of minutes |
| `history.py` | what each camera held before we touched it |
| `catalog.py` | what a camera says it has, read from `/all.cgi` |
| `docs/simulator-names.md` | where the variable names came from, and how to find more |
| `listening.py` | audio in, text out — whisper.cpp, vosk, or the keyboard |
| `vocabulary.py` | every word the deck needs to hear, built from the grammar |
| `arri.py` | the only file that knows ALEXA's API |

Everything above `arri.py` is camera-neutral, so a second brand is another
backend beside it rather than a rewrite.

## The rules it works by

**It cannot format a card, stop a take, or reboot a camera.** Seven
variables are writable — white balance, tint, ASA, shutter, the two SDI
processing paths and the sensor flip — and the check sits in `arri.py` at
the last point before the wire, underneath the grammar, the outcomes and
anything taught on the day. None of them can widen it; a new variable has
to be added there on purpose. Recording, media and power are absent by
design: a voice command mishears, and a formatted card does not come back.

**Nothing sends without a confirmation.** Every change hits the live
picture, and in ProRes white balance is baked into the image. Hear it,
show it, confirm it, then send.

**Anything uncertain is refused out loud.** Two numbers in one breath is
two commands run together, not an eight-thousand-kelvin one. A number on
its own is read the way a crew means it — a colour temperature inside the
white balance range, an EI below it — and refused when it is neither.
Values outside a camera's range, and enum options the camera would refuse,
never leave the deck.

**The viewfinder is the operator's picture.** Every overlay exists on each
output under a different name, so an outcome has a target: "show me log"
is the monitors, "log in the evf" is the finder, "center dot everywhere"
is both. The finder is never touched unless it is named — and the sensor
flip, which is the recorded image, refuses an output rather than pretending
to have one.

**"Go back to the look" is undo, not a value.** A look can be a LUT or
something the DP built that morning, so the only safe way back is what the
camera actually held — read before every write, kept in `history.py`.

**The camera describes itself.** `/all.cgi` publishes every variable with
its type, range and selectable options, so the vocabulary comes from the
camera rather than a hand-written table:

```bash
python3 -m aicam --learn 10.2.2.200 --save snapshots/alexa35.json
```

What the camera does *not* publish is what people call each setting. That
is the alias table in `catalog.py` and the phrases in `intents.py`, and it
is tuned by ear.

## Presets: a setup worth a name

Switching between anamorphic and spherical is three or four settings on
every camera, twice a day. What those settings ARE depends on the job — one
job it was squeeze factor, framelines and master magnification; the next it
might be the sensor mode — so a preset is captured from a camera somebody
has already set up right, rather than described in code:

```
save "spherical" from A          ← A is set up spherical now
                                 ← set A up anamorphic, in the menu, by hand
save "anamorphic" from A
B camera go anamorphic           → B: ok — 10 settings
```

`presets` lists them, `forget preset "…"` removes one. They live in
`~/.config/aicam/presets.json`, and a `"capture"` list in that file narrows
what a preset takes — a job that never touches the sensor mode should not
have a preset quietly carrying one. It can only narrow: what may be
captured at all is fixed in `presets.CAPTURABLE`, because that list is also
what the program may write.

Applied one setting at a time, in order, because the camera cares — a
squeeze factor is chosen against a sensor mode — and because a refusal
halfway through should name the setting that was refused. What each camera
held is read first, so "undo that" still works on all of it.

## Teaching it a phrase on the day

Something gets said, it gets refused, and nobody is stopping the shoot for
a code change:

```
A camera give me the log
  ignored: no value heard
  teach it:  learn like "show me log"
learn like "show me log"
  “A camera give me the log” now means “show me log” → A → SDI 1+2 → LogC
B camera give me the log
  B → SDI 1+2 → LogC
```

A taught phrase is stored as text in `~/.config/aicam/phrases.json`,
outside the repo, and works by rewriting what was heard into a phrase that
already parses — so it covers settings, outcomes and undo alike, and can
never reach anything the grammar could not already do. The camera name is
stripped when teaching, so a phrase taught to A works for B.

`taught` lists them, `forget "…"` removes one.

It refuses to learn a phrase that already means something, or one shorter
than two words. Teaching is typed, never spoken, so a phrase said to a
camera cannot redefine the language.

**With vosk, a taught phrase only works if its words are in the
recognizer's vocabulary** — vosk already reported that it has never heard
"ei" or "logc". whisper has no such limit. That tradeoff is unresolved.

## Addressing

One crew's convention is the default, not a rule: the camera letter is the
address, A at `10.2.2.200`, B at `.201`, C at `.202`. Change it with
`--subnet` and `--first`, or point one camera anywhere with
`--camera A=host`.

## Push to talk

```bash
VOSK_MODEL=models/vosk-model-small-en-us-0.15 .venv/bin/python -m aicam --listen \
  --cameras A,B,C --camera A=127.0.0.1:9001 --camera B=127.0.0.1:9002 --camera C=127.0.0.1:9003
```

**Hold space, talk, let go.** A terminal delivers characters and never says
a key was released, so the release is inferred: a held key auto-repeats, and
when the repeats stop, it was let go. The first repeat is delayed by up to
half a second, so a press that never repeats falls back to a toggle — press
to start, press again to stop — and `--toggle` forces that. Either way the
deck's real button, which does report press and release, changes nothing
else. It says what it heard and what it would
do; **return** sends it, **any other key** drops it, **q** quits. Add
`--dry-run` to talk to nothing, or `--keep-clips` to keep the recordings and
grow the test set from real use.

`--no-confirm` sends the moment it understands, which is two keystrokes
shorter and the way it will actually get used. The safety then rests on the
undo: every write reads the old value first, so "undo that" is a sentence
away, and a misheard command costs the seconds it takes to say it. Worth
keeping the confirm for anything baked into the recording — white balance
in ProRes especially — and dropping it for the rest.

A terminal cannot see a key being released, so the same key starts and
stops. The deck's button will press and release; everything either side of
that is the same code.

Use vosk here. whisper.cpp through its command line binary reloads its model
on every clip — 24 seconds, the first time we measured it — so it belongs in
the benchmark until it runs as a resident server.

## Which speech engine — measured, on 59 clips of one voice

| | commands right | per clip |
| --- | --- | --- |
| Apple, taught our phrases | **56/59** | 0.13s |
| vosk, held to our grammar | 52/59 | 0.03s |
| Apple, untaught | 49/59 | 0.15s |

Apple's recogniser is better **once it has been told what gets said here**.
Untaught it writes "essay" for ASA, "Hey camera" for A camera and "tent"
for tint, and lands behind a 40MB vosk model. `SFSpeechLanguageModel` takes
the 1,412 sentences `tools/make_phrases.py` generates from the same tables
the grammar parses — so what it expects to hear and what this can
understand cannot drift apart — and that is worth seven commands.

Run it again after changing anything:

```bash
python3 tools/make_phrases.py > /tmp/phrases.txt
open tools/swift/CustomLM.app --args /tmp/heard.tsv /tmp/phrases.txt "$PWD"/clips*/*.wav
python3 tools/score_transcripts.py /tmp/heard.tsv clips clips-2 clips-asa
```

## Which speech engine

Undecided on purpose. `listening.py` gives whisper.cpp and vosk the same
shape, so the board picks the winner with a stopwatch rather than an
argument:

```bash
python3 tools/record_phrases.py clips/        # say them, in a real room
python3 tools/bench_listen.py clips/ -v       # race whatever is installed
```

Engines are found through the environment and skipped when absent:

```
WHISPER_BIN=/usr/local/bin/whisper-cli WHISPER_MODEL=models/ggml-tiny.en.bin
VOSK_MODEL=models/vosk-model-small-en-us-0.15
```

The benchmark scores **commands, not words**. "Five thousand six hundred"
and "5600" are different transcripts and the same command; a word error
rate would rank them apart and tell you nothing useful.

On a Mac, against 38 fresh clips of real speech (`clips-2/`), whisper.cpp
gets 33/38 and vosk 34/38, at 0.23s and 0.03s a clip. The nine clips in
`clips/` are a regression test, not a score — the parser was fixed until
they passed.

The remaining misses are the engines losing words, not the parser: "800 asa
on a" came back as "A hundred A's A on A."

**A camera that is not on the job is refused.** vosk heard "A camera" as "e
camera" and would have sent 5600 K to a camera nobody named — the worst
thing this can do. `--cameras A,B,C` refuses anything outside the job, and
a camera name that was said but lost ("tungsten on **i**") is refused rather
than falling back to whoever was addressed last.

Getting there took five parser fixes, all of them driven by what the
engines actually wrote (`tests/test_grammar.py`, WhatTheEnginesActuallySaid):
whisper punctuates ("EI-1600", where the hyphen must not read as a minus),
both engines hear "tint" as "ten", vosk spells letters as words ("camera
be", "see camera e i"), reads 709 as "seven oh nine", and writes "go back
two the look".

The whole language is about a hundred words, listed by
`vocabulary.words()` and built from the grammar itself, so it cannot drift.
An engine that accepts a grammar is held to exactly that list — it cannot
invent a "thousand" in the middle of a white balance.

## Known to be wrong

**"Show me log" writes the wrong variable.** `SDI1Processing` is the kind of
feed a connector carries (clean, processed, clone); the log-versus-look
choice is `SDI1Gamma`, and 709 is `SDI1ColorSpace`. Found by scraping
ARRI's simulators properly rather than grepping them by hand.

[docs/processing.md](docs/processing.md) has the whole table, what is still
unknown (how the CGI encodes those values, and which variable an ALEXA 35
honours), and the two-dump diff that settles it on a real camera.

## Untested, and known to be

**A sequence is not confirmed.** A preset sends several writes in a row and
moves on as soon as the camera answers — and `{"result": 1}` means accepted,
not applied. The fake cameras answer instantly, so the code looks right and
proves nothing. Sensor mode is the one to be suspicious of: it is slow, and
it changes which squeeze factors and framelines are even valid.

[docs/sequences.md](docs/sequences.md) has what to measure on a real camera
and what to build depending on the numbers.

## Deliberately not built

**Shutter as a time.** The camera can take an exposure time instead of an
angle (`ExposurePreferenceIsAngle`), and "one forty-eighth" could be
converted — but only using the camera's current frame rate, since 1/48 is
180° at 24fps and 172.8° at 25fps. Shutter is called out in degrees on this
crew's sets, essentially always, so the conversion would be a real chance
of being wrong in exchange for a phrase nobody says.

**Monitor-only flip.** `MonitorFlipMode` exists, but "flip it" has to mean
exactly one thing, and low mode settled that it means the recorded image.

## Not done yet

- The winner of that race is not wired into the CLI yet: it is still typed,
  not spoken
- `show me log` and `back to the look` write `SDI1Processing` /
  `SDI2Processing`, from ARRI's menu simulator (see
  [docs/simulator-names.md](docs/simulator-names.md)) — right namespace,
  but unconfirmed on a real camera until someone dumps one
- false colour, ND, shutter and frame rate have names but no phrases yet
- Enum settings need spoken names for their options (ND, frame rate,
  shutter angle)
- Guarded settings — record, format, sensor mode — are flagged but the
  policy is undecided
- `B camera match A`: read one camera, write another

## The clip sets

| | what it is |
| --- | --- |
| `clips/` | the first nine. A regression test, not a score: the parser was fixed until they passed |
| `clips-2/` | 38 fresh phrases, the honest number — whisper 33/38, vosk 34/38 |
| `clips-asa/` | ASA vs ISO vs EI, four ways each |

`clips-asa/` answered a question worth writing down: **the word makes no
difference** — all three came through on both engines. What failed was the
number, and only when it came first ("800 iso on a" → "e hundred iso on
a"). Every clip with the number at the end passed on both engines.

So the short form is fine for a colour temperature, where the number is
distinctive, and shakier for an ASA, where it is small and common.

```bash
WHISPER_BIN=$(which whisper-cli) WHISPER_MODEL=models/ggml-tiny.en.bin \
VOSK_MODEL=models/vosk-model-small-en-us-0.15 \
.venv/bin/python tools/bench_listen.py clips-2/ --cameras A,B,C
```

---

## Changelog

### 0.3.0

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

#### What is still not right

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

### 0.2.0

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

#### What is still not right

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

### 0.1.0

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

#### What is not right yet

- **"Show me log" and "show me 709" write the wrong variable.**
  `SDI1Processing` is the kind of feed a connector carries; the
  log-versus-look choice is `SDI1Gamma` — and it is spelled differently on
  every camera generation. Everything else on the SDI side works; this one
  needs a real camera to confirm how the values are encoded before it is
  changed. See `docs/processing.md`.
- **A sequence is not confirmed.** A preset sends its writes one after
  another and moves on when the camera answers, and that answer means
  accepted rather than applied. See `docs/sequences.md`.
