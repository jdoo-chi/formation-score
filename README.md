# Formation Score

**Turn choreography into music.** Upload a dance practice video or a stage cam, and Formation Score tracks every dancer, reads where they stand and how their bodies move, and turns the choreography into a piece for strings and piano. It plays back in sync with the video and exports sheet music (MusicXML), MIDI and the dancers' stage coordinates (CSV).

Everything runs in the browser. Videos never leave your computer.

![Formation Score playing the built-in sample choreography](docs/screenshot.png)

---

## What's new

### Latest update

- **Ensemble arrangement (new default).** Music now comes from the dancers' bodies, not just their floor positions. Sharp accents in the arms and legs ("hits") become notes on the beat. The dancer at centre stage plays the lead, and raising their arms pushes it higher. Each new formation moves the harmony to the next chord, and when most of the group hits together, low piano and timpani hit with them.
- **Sampled strings and piano.** Violin, viola, cello, contrabass, pizzicato, harp, piano, timpani and a string section, rendered from the GeneralUser GS soundfont. The basic synth and its drum-like sounds are gone from this arrangement.
- **Pose detection.** The detector is now YOLOv8n-pose, which finds 17 body keypoints per dancer as well as their position.
- **Moving cameras.** A new **Camera moves** option in Analyze follows broadcast, high-angle and crane cams. It tracks the background (dancers masked out) with OpenCV.js and maps every frame back onto the first one, so dancers stay on the floor plan while the camera pans and zooms.
- **Classic arrangement kept.** The original one-line-per-dancer mapping is still available under Map → Arrangement → Classic.

### Earlier updates

- **Tracking recovers lost dancers.** A dancer hidden behind others is picked up again when they reappear, instead of drifting off the stage.
- **Fixed blank page on Windows.** The page now declares UTF-8, so it no longer breaks when served by Python's built-in server on Windows.
- **Clearer video errors.** If a video can't be decoded (often HEVC/H.265 from phones), the app says why and how to fix it.
- **Mapping editor (Classic).** Choose which movement drives pitch, dynamics and note density; reverse any of them for mirrored practice videos.
- **Score cursor.** A playback line follows the music on the rendered score.
- **Faster analysis.** Graphics-card (WebGPU) inference where available, with automatic CPU fallback, and three detector sizes (Fast, Balanced, Precise).
- **Redesigned workspace.** A five-step workflow (Upload → Analyze → Map → Compose → Export), a top-down trajectory view, a formation feature strip, a piano roll, and typeface settings.

---

## Run it locally

You only need **Python 3** (to start a tiny local web server) and a recent **Chrome or Edge**. Nothing to install.

1. Download or clone this repository.
2. Open a terminal in the project folder.
3. Start a server:

   **Windows (PowerShell or Command Prompt)**
   ```
   python -m http.server 8000
   ```
   If `python` isn't found, try `py -m http.server 8000`.

   **macOS / Linux**
   ```
   python3 -m http.server 8000
   ```

4. Open **http://localhost:8000** in your browser.

The terminal will just say `Serving HTTP on ... port 8000` and wait. That means it's running. Leave that window open while you use the app, and press `Ctrl + C` in it to stop.

> **Don't double-click `index.html`.** Opening it straight from disk (`file://`) blocks the model and instrument downloads, so the app can't analyze anything. Always go through `http://localhost:8000`.

The first analysis downloads the pose model (~13 MB, from this folder) and the onnxruntime engine from jsDelivr, so you need an internet connection. The **Camera moves** option loads OpenCV.js (~11 MB) from jsDelivr the first time you use it.

### Publish with GitHub Pages

Push the repository, then go to **Settings → Pages**, choose **Deploy from a branch**, select `main` and `/ (root)`, and save. Your app will be live at `https://<your-username>.github.io/<repo-name>/` after a minute or two. All files are under GitHub's size limits.

---

## How to use it

1. **Upload** a video. Best results: a fixed camera with the whole floor in view (dance practice videos, full-stage fancams). MP4 (H.264) is the safest format.
2. **Analyze.** Drag the four numbered handles onto the floor, as seen from the audience: 1 back-left, 2 back-right, 3 front-right, 4 front-left.
   - Place them as a rectangle facing the way the dancers face, not necessarily the physical corners of the stage. On a stage filmed from a corner, the real corners give a tilted floor plan.
   - Keep them inside any mirrored wall so reflections aren't counted as dancers.
   - For broadcast or high-angle cams, turn on **Camera moves** and set the corners on the first frame of the range.
   - Choose **12 fps** for the sharpest rhythm, set the number of dancers if you know it, and press **Run analysis**.
3. **Map.** Pick the arrangement (Ensemble or Classic), the key and mode, and set the tempo: tap **Tap** along with the music, then stop on the first downbeat and press **Set at playhead**. For slow songs, doubling the tempo catches faster hits.
4. **Compose.** Press play (or the space bar). Toggle the layers, switch between the piano roll and the score.
5. **Export.** Download MusicXML (MuseScore, Dorico, Sibelius, Finale), MIDI (any DAW) and the positions CSV.

---

## How movement becomes music

### Ensemble (default for analyzed videos)

| Movement | Music |
|---|---|
| A sharp accent in a dancer's arms or legs | a note, snapped to the beat grid of the tempo you set |
| The dancer nearest centre stage | plays the lead, in their own instrument |
| The lead raising their arms | the lead goes higher |
| Other dancers, stage left to right | lower to higher chord tones |
| A new formation | the next chord (I–V–vi–IV in major, i–VI–III–VII in minor) |
| Symmetric / scattered / in a line | triad / add9 / sus2 |
| Most of the group hitting together | low piano octaves and timpani |
| Moving into a new formation | a harp run into the new chord |

Dancers cycle through violin, piano, viola, harp, cello and pizzicato; a string section and contrabass hold each chord. A formation held for more than four bars moves the harmony on by itself. Ensemble needs body poses, so the built-in sample (which has none) plays Classic.

### Classic

One line per dancer from floor position alone. In the Map step, each of **pitch**, **dynamics** and **note density** can follow left ↔ right, back ↔ front, speed, distance from centre, or stay fixed, and can be reversed.

| Layer | Movement | Music |
|---|---|---|
| Voices (one per dancer) | left ↔ right | pitch (stage width = 1–3 octaves) |
| | back ↔ front | dynamics |
| | crossing a pitch zone, or moving on the beat | new note |
| | standing still | held note |
| Formation (one chord per beat) | centroid left ↔ right | chord root (centre stage = tonic) |
| | mirror symmetry | triad / seventh / cluster |
| | dancers in a line | stacked fourths |
| | spread | voicing width |

---

## How it works

- **Detection**: YOLOv8n-pose (ONNX) on onnxruntime-web 1.30.0, using WebGPU when available and WebAssembly otherwise. Each dancer's foot point is the bottom centre of their box.
- **Floor plan**: a homography from the four corner handles to a unit square.
- **Moving cameras** (optional): ORB features on the background, matched frame to frame and re-anchored to the first frame every third frame, with a RANSAC homography (OpenCV.js 4.12). Roughly doubles analysis time.
- **Tracking**: fixed dancer count, Hungarian assignment with constant-velocity prediction, and re-acquisition of dancers who were hidden.
- **Body hits**: peaks in limb acceleration (wrists, elbows, knees, ankles), normalised by body height.
- **Sound**: Web Audio. Ensemble plays sampled instruments through a convolution reverb; Classic uses simple synth voices.
- **Score**: OpenSheetMusicDisplay 2.2.1.

## Project structure

| Path | What it is |
|---|---|
| `index.html` | The app, ready to serve |
| `src/page.html`, `src/style.css` | Markup and styles |
| `src/app.js` | Interface, analysis loop, audio, export |
| `src/core.js` | Geometry, tracking, Classic composition, MusicXML and MIDI writers |
| `src/arrange.js` | Ensemble arrangement |
| `samples/` | Instrument samples (MP3) and their licence |
| `yolov8n-pose.onnx` | Pose model weights |
| `ort-wasm-simd-threaded.wasm` | onnxruntime-web CPU engine |
| `build.py` | Rebuilds `index.html` from `src/` |
| `docs/screenshot.png` | Screenshot for this README |

### Editing the code

`index.html` is generated. Edit the files in `src/`, then rebuild and refresh the browser:

```
python build.py
```

(`python3 build.py` on macOS / Linux.)

### Custom typefaces

The **Aa** button lets each viewer pick a preset, an installed font, or a font file; the choice stays in their browser. To change the default for everyone, add a `@font-face` rule at the top of `src/style.css`, put the family first in `--font-ui` (or `--font-mono`), and rebuild. Check that the font's licence allows web embedding.

---

## Troubleshooting

| Problem | What to do |
|---|---|
| The page is blank or nothing responds | Make sure you opened `http://localhost:8000`, not the file. Hard-refresh with `Ctrl + Shift + R`. |
| Choosing a video does nothing | The browser can't decode it. Convert to MP4 (H.264), e.g. with HandBrake, or record with the phone's "Most Compatible" setting. |
| Only one or two dancers are tracked partway through | The camera moved. Turn on **Camera moves**, set the corners on the first frame of the range, and analyze again. |
| More people are found than there are dancers | Mirrors or people at the edge. Move the corner handles inside the dance area and set the dancer count by hand. |
| Dancers swap labels | Set the dancer count by hand and analyze at 12 fps. Fast crossings can still swap identities. |
| Every chord sounds the same kind | The floor plan is tilted. Re-place the corners as a rectangle facing the dancers' front. |
| Analysis is slow | Use **Fast** detector size, a shorter range, or Chrome/Edge with WebGPU. The Analyze step shows which engine ran. |

Tested in desktop Chrome. Other browsers may work but haven't been tested.

---

## Licence

This project is released under the **GNU AGPL-3.0** (see `LICENSE`), because it ships the YOLOv8 pose weights, which Ultralytics licenses under AGPL-3.0. For a closed-source or commercial release, get an Ultralytics enterprise licence or swap in a detector under a permissive licence.

Third-party components:

- **YOLOv8n-pose** weights: Ultralytics, AGPL-3.0
- **Instrument samples**: rendered from GeneralUser GS by S. Christian Collins, free for any musical use (`samples/LICENSE-GeneralUser-GS.txt`)
- **onnxruntime-web** (MIT), **OpenCV.js** (Apache-2.0) and **OpenSheetMusicDisplay** (BSD-3-Clause), loaded from jsDelivr
- **Geist** and **Geist Mono** (SIL OFL), loaded from Google Fonts

Videos you analyze stay on your machine. If you share results made from someone else's footage or music, credit the source and check the platform's rules.
