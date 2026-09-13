# Chordially

Upload a page of printed violin sheet music. Chordially splits it into practice
passages, rates each passage from 0.0 to 10.0, colors the page by difficulty
from light green to near-black, and tells you how to practice the passage you
select — including rhythm variations built from that passage's own notes.

**Live demo:** https://chordially-azure.vercel.app — open the example score or
import a MusicXML file. Reading scanned pages needs a local install; see
[Hosted version](#hosted-version).

## What it does

- **Reads your music.** Upload a PDF, PNG or JPEG of one printed page, or import
  MusicXML (`.musicxml`, `.xml`, `.mxl`). Scans are recognized locally with
  Audiveris and shown re-engraved, and the page says so. MusicXML is read
  exactly from the file and engraved with Verovio — on the first page of
  Mozart's K. 156, all 145 measures are read and rated.
- **Groups it into musical units.** Notes form phrases; adjacent phrases that ask
  for the same kind of work form a practice passage, which splits where a
  printed key or meter change, a real step in difficulty, or a change in the
  main demand says the work has changed. Short trouble spots are marked inside a
  phrase only where one genuinely stands out.
- **Rates difficulty and shows its working.** Ratings come from features read off
  the notation: note rate, fine subdivision, displaced accents, accidentals
  outside the key, register above first position, wide leaps, double stops,
  bowing demand, rhythmic complexity, key remoteness, and estimated string
  crossings and position changes. Tempo comes from the printed mark or tempo
  word, can be overridden, and changes every rating. The weights behind a rating
  are shown beside it.

  | Rating | Label |
  |---|---|
  | 0.0 – 1.9 | Beginner-friendly |
  | 2.0 – 3.9 | Advanced Beginner |
  | 4.0 – 5.9 | Competent level |
  | 6.0 – 7.9 | Expert level |
  | 8.0 – 10.0 | Extremely hard |

- **Colors the page.** Each passage is highlighted in its difficulty color, on a
  scale running light green → yellow → orange → red → near-black. Measures that
  could not be read stay hatched and unrated — never shown as easy.
- **Teaches the passage you pick.** Click anywhere in a passage to select it. The
  sidebar explains what makes it hard and offers exercises chosen from what the
  notation shows — complementary rhythms (computed exactly, so every variant
  lasts as long as the written notes), counting the subdivision, slow practice
  with one stated goal, looping the hard join, and more. Each exercise says how
  to do it, what to listen for, and how to return to the music, and cites
  whether its support is teaching practice, research, or a heuristic.
- **Lets you disagree.** Split or merge phrases and passages; ratings and advice
  follow the change.

## Limits

- Ratings are heuristic estimates, not measured proficiency or validated
  examination grades. On the bundled Wohlfahrt page they agree with the order
  Wohlfahrt published his studies in, at 60, 90 and 160 BPM — but that is one
  pair of studies on one page, and the harder study has only a small sample of
  readable measures.
- Recognition checks that each measure's durations add up to the meter and that
  its notes are in the violin's range. It does not verify pitch.
- Fingerings, string choices and shifts are never stated as fact when the
  notation does not print them.
- One page per upload. Phrase boundaries can be placed only at barlines.
- Practice progress and settings are not saved, and an analysis lasts only as
  long as the server process.

## Hosted version

The live demo runs on Vercel; pushes to `main` deploy it. It differs from a
local run in three ways:

- Scanned pages cannot be read there, because Audiveris is a Java application
  and is not installed. The upload says so. The example score and MusicXML
  import work.
- Vercel refuses uploads larger than 4.5 MB.
- An uploaded analysis lives in one server instance's memory, so its link can
  stop working later.

## Run it locally

Python 3.14.

```bash
pip install -r requirements-dev.txt
python -m uvicorn src.app.main:app --reload       # http://127.0.0.1:8000
```

Tests:

```bash
python -m playwright install chromium             # once, for the browser tests
python -m pytest -q                               # everything
python -m pytest tests/unit tests/integration -q  # skip the browser suite
```

There is no build step: the client is plain ES modules served as-is.

### Reading scans

Scan recognition runs through **Audiveris 5.11**, a Java application that
`pip install` does not cover. The Windows console MSI unpacks without
administrator rights and carries its own JDK:

```
msiexec /a Audiveris-5.11.0-windowsConsole-x86_64.msi /qn TARGETDIR=work\audiveris\extracted
```

Chordially looks for it at `work/audiveris/extracted/Audiveris/Audiveris.exe`,
then at `PRACTICEMAP_AUDIVERIS_EXE`, then on `PATH`. Without it, a scan upload
fails with a recovery action; MusicXML import and the example score still work.

### Configuration

Copy `.env.example` to `.env`. Everything is optional.

| Variable | Purpose |
|---|---|
| `PRACTICEMAP_AUDIVERIS_EXE` | Path to the Audiveris executable |
| `PRACTICEMAP_SCAN_ENGINE` | `audiveris` (default) or `vision` |
| `PRACTICEMAP_ANTHROPIC_API_KEY` | Only for the `vision` engine, which sends page sections to Anthropic's API, and for rebuilding the example fixture |
| `PRACTICEMAP_WORK_DIR` | Where rendered pages and caches go (default `work/`) |
| `PRACTICEMAP_SYNC_JOBS` | `1` runs analysis inside the upload request (automatic on Vercel) |

## How it's built

- FastAPI and Jinja2 on the server, plain JavaScript in the browser, Pydantic
  models validating every recognition result at the boundary.
- Measure geometry is measured, not estimated: OpenCV staff and barline
  detection on scans, and measure boxes read from Verovio's SVG for MusicXML.
  Overlays are positioned in page percentages, so they stay aligned through zoom
  and resize.
- Timing arithmetic uses exact fractions throughout.

```
src/app/        web app and templates
src/features/   segmentation, difficulty, practice, score viewer
src/server/     recognition adapters and the analysis pipeline
src/schemas/    validated domain models
src/content/    practice techniques and their sources
src/static/     CSS and JavaScript
tests/          unit, integration and browser tests
fixtures/       example scores and reference analysis (see fixtures/README.md)
scripts/        fixture rebuild tools
```
