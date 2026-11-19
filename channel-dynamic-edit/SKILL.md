---
name: "channel-dynamic-edit"
description: "Use together with premiere-auto-edit when editing YouTube-channel videos: importing a recording day, dynamic full-face/split layouts with a variable ratio, real screen demos, long horizontal screen+camera videos, and cutting shorts from a long video."
---

# Channel dynamic edit (add-on to premiere-auto-edit)

Load `premiere-auto-edit` first and follow it, including its SETUP (config, permissions, bootstrap). This skill ADDS the channel rules below and wins where the two differ. Read `<root>/_tools/edit-config.json`; this skill uses two extra keys, asked once together with the S1 questions if missing:
```json
{ "channel_handle": "@yourhandle", "shorts_theme": {"bg_from": "#1A2C3D", "bg_to": "#0A141E", "accent": "#4ADE80"} }
```

## Standing rules (never ask)
1. DYNAMIC VIDEO: never a talking head for long stretches. From the first seconds alternate full face -> split screen (panel on top showing what the speaker is talking about) -> full face -> split ... until the end, one split segment roughly every 10-20 s, each matching the spoken topic.
2. SPLIT RATIO: never default to 50/50. Size the screen part by what it shows: usually 40% screen / 60% face; 35/65 or 30/70 for wide/short content; 50/50 only for tall content.
3. FACE NEVER CUT: in every split the whole head stays below the seam; verify on exported frames.
4. REAL SCREENS FIRST: when the speaker talks about a product or feature, show the real screen (recording or capture, section F). Remotion for concepts.
5. CONCEPTS = ANIMATED DIAGRAMS, not static lists: integrations, version control, databases, caching, deploys, APIs etc. get arrows, moving packets, timelines (premiere-auto-edit 18.1).
6. OFFICIAL LOGOS everywhere a product/service is named (section E). Never emoji for brands. If a logo is missing, find it and say where it came from.
7. FACE ZOOM: full-face clips may alternate zoom for dynamism (e.g. 110 <-> 120), but not too often. When switching to split, KEEP the zoom and x of the full-face shot right before it; only move the face lower. No separate split zoom.
8. OPENING: start with the main clip's own first sentence; remove stumbles, noise, false starts. Zoom-out on the face at the start with a whoosh. In the first 60-90 s cut even small silences.
9. CAPTIONS: bottom in full-face parts, on the seam in split parts (in shorts too; never at the very bottom where platform UI covers them).
10. Every feedback round = a new cloned sequence or new asset names (previous kept). Preview only when asked. Never revert the creator's manual changes. Warn explicitly about anything missing (logos, caption styles, recordings, SFX).
11. TEXT SIZE: titles/animations as big as possible but NEVER covering the face or the screen content being shown.
12. Every pop-up animation gets an SFX; transitions where shots change.
13. Minimal talk. When the creator says "do it and tell me when done", work autonomously and report briefly at the end.
14. PRIVACY: premiere-auto-edit 8.7 on every frame (emails, tokens, phone numbers, other people's names, private tabs), including the first and last frames of each blur, unless the creator explicitly allows an item.

## A. Getting a recording day
- Source: the creator's cloud photo library (e.g. Google Photos) or a local folder. For a cloud library, use the session's browser on the computer that holds the workspace; check which account is signed in and ask the creator if it's not the one they named. Select the day's new clips (e.g. Google Photos "Recently added" -> select the day's group -> download as zip), wait until the download finishes (`.crdownload` gone).
- The device mount is slow: unzip a few files per call (`timeout 165 unzip -n -q zip f1 f2 ...`) into `<root>/<date>/`.
- Ask once how many videos the day contains and which clips belong together. 1-5 s clips are usually junk; short hook retakes are retakes.
- Layout: raw clips in `<date>/`; per video `<date>/<VID NN topic>/{remotion,captions}`; `<date>/_work/VID NN/`, `_work/transcripts/`, `_work/previews/`; exports `<date>/export/` (superseded to `export/_old/<reason>/`, shorts in `export/shorts/`). VID numbering continues across days. Final sequences live inside the video's bin.
- ONE PREMIERE PROJECT PER VIDEO: every long video and its shorts get their own project `<date>/<VID NN topic>/VID NN topic.prproj` containing only the video bin + `Shared` (sfx). To split an old video out of a big project: back up the project to `_backup/`, check the video's sequences use nothing outside its bin except `Shared`, `app.project.saveAs(new path)`, move every other root item into a temp bin and `deleteBin()` it, verify the sequence count, save; then remove the video's bin from the original project and save.

## B. Transcribe the whole batch
- ffmpeg each clip to mono 64k mp3 in parallel; transcribe with Scribe (premiere-auto-edit section 5) in parallel, word timestamps, language from config. Apply the workspace fixes list.

## C. Windows + Premiere notes
- The Premiere MCP may run as a desktop extension; `execute_extendscript` returns a value only with an explicit `return`. Long scripts: `return $.evalFile(new File("C:/.../x.jsx"))`. Keep each call short (place, fix slivers, export as separate calls); a timeout may still have finished the work: check state before re-running. Globals persist between calls: reset option globals in each call.
- If every call times out: a modal may block (import progress, "File Import Failure"): close it (computer use, Escape), then Stop/Start the bridge in the MCP Bridge panel; retest with `return app.project.activeSequence.name;`.
- Paths: forward slashes for `new File`, backslashes for `exportFramePNG`/`encodeSequence`. Avoid "/" and "." in sequence names (Premiere truncates them). Find bins by exact name.
- If the Media Encoder queue never starts: export with `seq.exportAsMediaDirect(out, preset, app.encoder.ENCODE_IN_TO_OUT)` in a jsx loop logging each item to a txt you poll; afterwards close Media Encoder so its old queue does not write duplicate files later.
- Frame checks: export the sequence and grab frames from the mp4 with ffmpeg; don't plan timing from `-ss` seeks into OBS .mkv files (inaccurate).
- SEQUENCE = SOURCE: 1080p footage -> 1080p sequence; 4K sources -> edit in 1080p 30 fps (no need to ask). Converting a 4K sequence: set 1080 settings and halve every Motion Scale incl. keyframes (positions are normalised, unchanged).
- FACE FRAMING in a vertical 1080x1920 sequence (k = Scale/100 when the clip fills at Scale 100): full face x = 0.5 + (0.5-faceX)*k, clamped to [1-k/2, k/2] so no black edge. Split: same Scale and x as the previous full clip; face centre on screen F = h + (1920-h)*0.33 (h = panel px); y = (F - (faceY-0.5)*1920*k)/1920, keep y*1920+960k >= 1920.
- WARMTH: Lumetri Temperature per video by comparing face R/B (premiere-auto-edit 4). One-sided audio -> Fill Right with Left.
- AUDIO: aim for about -12 to -14 LUFS with a Hard Limiter (max amplitude 0.99). Low-frequency pulsing/rumble -> high-pass ~100 Hz + gentle compression below 300 Hz into a new wav, swap A1 (re-sync in-points to V1). Enhance Speech is not scriptable.

## D. Split segments and Remotion
- Plan from the transcript. Panels are OPAQUE H.264 mp4 at real size (e.g. 1080x768 for 40%, 1080x672 for 35%) on V2 at Scale 100, Position (0.5, (h/2)/1920). Whoosh at split start/end.
- Keep a reusable Remotion generator in `_tools/remotion/` (container working copy e.g. `/home/claude/remo`): scene kinds list / flow / vs / shots / images / big / net (nodes, links, moving packets incl. blocked, orbits, zones, counters, badges) / timeline (commit history + rewind); per-video data files; a logo component (`logo:<key>` = official logo in a white circle with an accent ring); a root that renders panels plus 15 s chunks of transparent full-face overlays and captions. Concurrency 2.
- Check stills of every new scene before rendering; keep labels off lines, badges away from nodes.
- Transparent overlays: render ProRes 4444, convert to qtrle argb (`-g 100000`) in 10 s chunks so each file stays under ~25 MB.

## E. Official logos
- Sources in order: `@iconify-json/logos` (npm, full-colour logos: gmail, drive, notion, git, github, openai, claude, anthropic, gemini, google, meta, nodejs, languages, databases, OS, paypal, facebook, cursor, adobe apps...), `@lobehub/icons-static-svg` (AI brands), `simple-icons` (monochrome fallback), `raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/<name>.svg`. Save SVGs to `remotion/public/logos/<key>.svg`.
- Many CDNs/brand sites are blocked from the container and the device shell; npm and raw.githubusercontent usually work.
- Dark logos (GitHub, OpenAI, Anthropic...) need a white circle background.

## F. Real screen demos without a recording
- Capture with the session's browser: page zoom ~200% + narrowed column, zoomed screenshots of the region saved to disk at native size. For non-Latin text input use `document.execCommand('insertText', ...)`. Animate captures in a `shots` scene. Use test data only, never real customers' or private data; afterwards tell the creator which test chats/events/items to delete. If the browser blocks a site on every call, ask the creator to close the stuck tab.
- Anything that needs the creator's own screen (signed-in desktop apps, private dashboards) cannot be captured this way: list those videos at the end so the creator can record or share their screen.

## G. Delivery
- `device_commit_files`: files < ~26 MB, <= 50 files per call, and it silently does NOT overwrite: always commit under a NEW name and relink with `projectItem.changeMediaPath(newPath, true)`. Never overwrite a media file Premiere already uses.
- Export with "Match Source - Adaptive High Bitrate" with in/out 0 -> last V1 clip end (`ENCODE_IN_TO_OUT`). Move superseded exports to `export/_old/` before re-exporting. Check durations with ffprobe.
- Preview only when asked: 540x960 crf 28 to `_work/previews/`, `SendUserFile` with display "render". If the creator asks for full quality, send the export itself.

## H. Long screen-recorded tutorials (OBS screen with a round face bubble already in the recording)
- Recorded as several long parts plus a short intro recorded last. Do NOT transcribe the long parts fully: transcribe the first and last minute of each part to trim heads/tails; the middle stays uncut except long silences. Make the intro engaging and the ending clean.
- No subtitles on the long video unless asked. Don't raise volume on your own if levels are fine; ask once if unsure.
- Resource mentions ("the repo is in the description") go at the BEGINNING, before building starts.
- Layout: framed layout with motion graphics + captions only for the intro; after that the screen recording FULL FRAME with no motion graphics. Switch with a 0.5 s scale/position keyframe.
- SFX for animations around raw Level 0.04.
- ENDING: never cut what the speaker says after the thank-you (like/subscribe/goodbye); only remove long silences. Restored clips get the same gain as the rest.

## I. Long horizontal screen + camera videos (separate camera file)
- Sources: screen recordings (often 1080p60 with good audio: use as the voice) + phone camera (often 4K HEVC). Sync each pair once by audio cross-correlation (camera time = screen time - OFF); edit in 1920x1080 30 fps.
- LAYOUTS chosen per piece from the transcript:
  - F (full face): only while the speaker sits and talks to the camera.
  - B (screen FULL FRAME + face circle): the default whenever they aren't just talking to the camera. The screen is usually more important than the face. No side-by-side layouts.
  - D (screen only, face hidden): when text is small or the face would cover anything important.
- FACE CIRCLE: a true circle (no square frame), top-right (~270 px diameter at 1080p, centre about (1748,170)); zoomed out so there is space above the head; face centred. Move it (down/left/elsewhere) whenever it would cover something important; hide it if there is no free spot. Technique: symmetric Rounded Crop (L=R, T=B), Roundness 100 plus all four per-corner roundness properties at 100 (otherwise it renders as a rounded square); move the mask onto the face with the effect's Offset; then set Motion Position. Calibrate on 1-2 clips per camera, check exported frames, then apply to all.
- SCREEN: as large as possible so small text is readable on a phone. Zoom-ins are fine; never animate a zoom-out. Don't zoom so far that context disappears (which tab/app is acting, what was opened).
- OPENING: keep any standing intro shot brief; cut even small silences in the first 60-90 s.
- CUTS: remove long waits (renders, loading) and show the result ~1 s before it appears. Never chop words: if a cut damages wording, put back the original continuous clip. Verify speech cuts by transcribing the exported range.
- OVERLAYS: big readable pills and one-line lists, a glitch-style notice where the speaker points out a mistake, enlarged views of small UI. Every pop-up gets an SFX; transitions where shots change; no static face for long stretches. Text never covers the face or screen content.
- Scripting gotchas: QE razor can leave 1-frame slivers/overlaps (match clip starts with ~0.045 s tolerance, measure real gaps). `exportFramePNG` never overwrites: use new prefixes. Bridge timeouts often still execute: check state before re-running.

## J. Shorts cut from a long video
- WORKFLOW (premiere-auto-edit RULE #0):
  1. Analyse the whole transcript and decide how many logical, useful shorts it gives, ranked T1 (very useful) -> T2 (less useful) -> T3 (plain).
  2. Put the list in a doc/canvas artifact: number, tier, title, time range, hook, what is shown, and what extra recording would make it better.
  3. Without waiting, edit ONLY THE FIRST short in Premiere (inside the video's own project) and say it's done.
  4. The creator checks it (offer a preview, don't send automatically). After approval do the next 2-3, report, wait. Repeat.
  A new design or pipeline always counts as unapproved: show one short first.
- Files: `export/shorts/sNN TX - title.mp4`.
- FACE LAYOUT: the face goes in a circle ONLY when the source recording itself has a face bubble. With a separate camera file use the normal short layout: screen recording in the TOP part, face in the BOTTOM part (split sized by content, rule 2), full-face shots where the speaker only talks, captions on the seam, same face zoom across full <-> split.
- STYLE (from config `shorts_theme`): background gradient bg_from -> bg_to, accent colour for caption pills and rings. Vertical 1080x1920: title in white bold at the top; captions on the SEAM as an accent pill with dark text; screen as a rounded card (thin 15% white border). Footer INSIDE the safe zone (well above the bottom ~15% where platform controls sit): official YouTube logo + "Full video on YouTube: <channel_handle>" + the LONG VIDEO TITLE. Last 3 s the title may swap to the same call to action.
- Bubble sources: V1 screen with Motion keys from the zoom plan; V2 face cropped out of the bubble and placed in the circle; a mask image (dark circle with accent ring) keyframed over the original bubble inside the screen; V4 Remotion overlay chunks.
- ZOOM (every short): zoom into wherever the action is: hide app sidebars and zoom into the main content; when a modal opens zoom onto it; zoom out to full width only when both sides matter; in an IDE zoom onto the editor/terminal/panel in use; on mostly empty screens (chat app home) zoom onto the input box. Plan from 1 fps frames of the exported short; never cut text at the card edge; take switch times from the EXPORT (`scdet` on the card crop), not from source mapping (can be off by 1-2 s); keep the zoom window inside the frame.
- BEFORE/AFTER shorts (raw vs edited): if the recorded screen is horizontal, stack the two versions top/bottom, not side by side.
- A change to a static overlay area can be patched onto finished exports with ffmpeg (render the strip, overlay it, re-encode at the source bitrate), but then also update the overlays in the Premiere project.

## K. ffmpeg and delivery gotchas
- `device_commit_files` inserts a metadata box after `ftyp` in MP4s. A whole MP4 stays valid; when splitting large files into parts (< ~20 MB each) join them with plain `cat` on the device and do NOT strip the box. Check with an ffmpeg decode.
- ffmpeg input `-ss` + `fps` keeps pre-roll frames (~+0.9 s per segment on OBS/phone sources) and drifts sync in concatenations: use `trim=start=0:duration=d,setpts=PTS-STARTPTS` (+ `atrim/asetpts`) before `fps`.
- Device shell background jobs die when the call ends: keep each ffmpeg job under ~170 s per call.
