---
name: "nim-vox-video"
description: "Use whenever the user asks for a vox video, Vox / Fern / Johnny Harris style, видео в стиле Vox, paper-collage explainer, collage motion ad, or a short explainer/ad for tech, product, social, finance, history or documentary topics built from cutouts, paper, marker and kinetic type; also for a series of such videos. Produces script, VO, assets and finished MP4s assembled by a code engine. Needs a shell with Python and uv (coding agents such as Claude Code, Codex, Cursor). Not for single Seedance clips (nim-generate) or long multi-clip videos (nim-long-video-chain)."
metadata:
  version: "8.1.0"
---

# Vox Video (design first, then code)

An image model designs every scene as one dense collage frame. The same model hands back the frame's objects
on a chroma sheet and its empty background plate. Code finds where each object sits, lays the pieces back
exactly there, assembles them on the beat of the voice-over, gives objects the action their words ask for, and
draws every word, number, source and mark.

Reply in the user's language. Scripts, prompts and on-screen text in English unless asked. No em dashes.
Read reference files only at the step that needs them (named in each step).

## Pipeline

| # | Stage | Output | Gate |
|---|---|---|---|
| 1 | Brief, format, length; facts with sources | `facts.md` | ask max 1 question |
| 2 | Script (2.5 words/s), one hook word per sentence | `script.md` | user approves (skip if told) |
| 3 | VO: `generate_voice_sample` (same `voice_description` for every part) → `vo_pick.py --max-len` → `align.py` | `vo.wav`, `words.json` | length ≤ target |
| 4 | Art direction once per video or series (`references/art-direction.md`) | `art.md` | |
| 5 | Storyboard: one scene per phrase: words, hero, title, 6-10 objects, action per object | `board.md` | |
| 6 | Per scene: frame → sheet + plate → `pack_split` → `layerize` → `scene_build` (`references/worker.md`) | `scenes/sNN/` | `lay_check.jpg` |
| 7 | `timeline.json` (scenes, transitions, packshot; `references/timeline-schema.md`), `--check` | timeline | `layout OK` |
| 8 | `--gate` and `--stills scenes` | sheets | design gate PASSED |
| 9 | `--draft`, then final `--out` (`--jobs`) | MP4 | QA below |

## Cost and tools

- Before paid generation show `get_credit_balance` and the estimate once; one OK covers a whole batch.
- Find each model with `models_explore` and follow its live `generationContract`: GPT Image 2.5 Flare (design
  frames and their edits) and Nano Banana Pro via `generate_image`; Seedance via `generate_video`.
- Never poll generation status in the chat: fire a batch of `generate_image` calls, then
  `uv run scripts/nim_fetch.py <promptId>=<path> ...` waits and downloads them all. Upload local files once with
  one `media_upload` token for many files.
- Credits per scene: frame 6 + sheet 9 + plate 9 = 24 (GPT Image 2.5 Flare Medium, 2K). Person cutouts: free, local
  `cutout.py`. VO 6 per call. A 15 s ad with 6 scenes + code packshot: 175-230 cr.
- Seedance only for real motion inside a frame (a screen, pouring), Seedance 2 Fast 480p as a layer; never to
  assemble a scene.

## Rules that are never bent

- Real people only from supplied or freely licensed photos (record source and license). In a design frame a
  person is a plain flat mid-grey silhouette; the final uses a cutout of the real photo. Never generate, restyle
  or edit a real person's face. No crime devices on real people (bars, crossing out, mugshot grading).
  Opponents neutral and respectful.
- Real products only from real photos. Logos are never redrawn.
- Every on-screen claim is verified and gets a small source strip (outlet + date). Headlines and quotes verbatim;
  shorten rather than paraphrase. Disclaimers verbatim, regular weight, about 20 px per 1080 width.
- Code draws all text: titles, name plates, counters, source strips, quote cards and clippings (`typeset.py`),
  packshot, disclaimer. The image model never writes readable text.
- Every movement is explained by a word of its phrase. Otherwise the object only lands (assembly) and then
  breathes (living paper). No looping decoration, no random spins, nothing exits; the next plate covers.

## Design frame in one paragraph

GPT Image 2.5 Flare Medium, `resolution: 2K`, the video's aspect ratio. Prompt = layout (hero placeholder, 3
layers: background papers, middle objects, foreground objects; title space as blank yellow strips) + the STYLE
BLOCK and CLOSER from `art.md`. Objects big, overlapping, cropped by the frame edges, 6-10, every one a noun
of the phrase. Then two edits of the frame: sheet on flat magenta #FF00FF (split free with `pack_split --mode
key`) and the empty plate. Templates and prompt text: `references/design-frames.md`.

## Sub-agents

For one video: the coordinator does stages 1-5 and 7-9; workers (`Agent` tool, at most 4 at once) do one scene each
with the brief in `references/subagents.md`. For a series: one worker per video. Probe once that workers can
load the Nim tools; if not, the coordinator runs every paid call and workers only split, layerize and build.

Saving compute:
- Start production in a fresh chat; run workers on a cheaper model (Sonnet) with `references/worker.md` as their
  only reading.
- Look at images downscaled and only at checkpoints: `lay_check.jpg`, the gate sheet, the stills sheet, the
  final transition grid. Do not open every generated file.
- Workers write no build scripts: `scene_build.py` turns `lay.json` + the storyboard row into `scene_NN.json`.

## QA (yourself, before showing)

1. `--check` → `layout OK`, no `EMPTY PLATE` warnings.
2. `--gate` → every scene PASS (fill ≥ 0.55, biggest empty square ≤ 7% of the frame, match ≥ 0.45); open each
   `design | final` sheet.
3. `--stills scenes`: start, middle and end of every scene. Every cutout looked at on colour.
4. Frame grid around each transition.
5. Audio: -14 LUFS, speech clearly above the paper sounds (`audio.sfx_vol` default 0.64), the video ends ≤ 1.5 s
   after the VO.
6. Deliver with SendUserFile, plus credits spent and wall time.

## Bundled files

- `scripts/vox_engine.py`: engine (`--check [--fix]`, `--gate`, `--stills scenes`, `--draft`, `--out`, `--jobs`)
- `scripts/layerize.py`: design frame → placed layers, hero and title slots, draft scene
- `scripts/scene_build.py`: storyboard row + draft → finished scene fragment
- `scripts/pack_split.py`: sheet → items (`key`, `alpha`, `white`, `ink`, `engraving`, `green`)
- `scripts/nim_fetch.py`: waits for production generations and downloads them by promptId
- `scripts/cutout.py`: local cutouts of people and products (rembg)
- `scripts/typeset.py`: quote cards, clippings, source strips + highlight boxes
- `scripts/vo_pick.py`: one voice across VO parts; `scripts/align.py`: word timings
- `references/`: `worker.md` (all a scene worker needs), `design-frames.md`, `art-direction.md`, `actions.md`,
  `timeline-schema.md`, `subagents.md`, `cutouts.md`, `verticals.md`
- `assets/`: fonts and desk textures; `examples/grow-usa-ad2/`: a finished timeline with scene fragments
