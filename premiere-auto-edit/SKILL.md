---
name: premiere-auto-edit
description: "Fully automatic Premiere Pro edit of a talking-head video (vertical short by default): cuts, audio, transcript, captions, split-screen demos, Remotion explainer clips, After Effects motion graphics, sound effects, organizing and export. Use when asked to auto-edit a video in Premiere Pro."
---

# Premiere auto-edit (zero-intervention editing system)

Goal: take a talking-head recording and deliver a finished edit with as few questions as possible, like an experienced editor with taste, following ONE repeatable system (sections 0-18). The creator wants minimal talk: reply in the language they write in, short. Final message: what was done, anything uncertain, nothing else.

TWO KINDS OF SOURCE FOLDERS (decide per folder):
- (A) STANDARD: one camera video (+ optional demo recordings / screenshots placed by meaning). Sections 0-14 apply as written.
- (B) TWO RECORDINGS OF THE SAME TAKE: a camera file and a screen recording of the same speech. Section 15 decides sync, which audio and denoise; everything else follows the standard sections.

SECTION MAP: SETUP (S1 config, S2 permissions, S3 bootstrap) | STANDING MODE | RULE #0 | RULE #1 | Workspace layout | 0 Preflight | 1 Audio | 2 Cuts | 3 Noise + level | 4 Colour | 5 Transcript | 6 Captions | 7 Caption style | 8 Overlays (8.1-8.8) | 9 Brand pack, zoom-out, SFX | 10 After Effects | 11 Key-phrase animations | 12 Finish + cleanup | 13 Creative visuals | 14 Pitfalls | 15 Two recordings | 16 Preview + export | 17 Face framing, face-only, revisions | 18 Remotion.

---

## SETUP (first run in a new workspace, then only when something is missing)

### S1. Creator config (ask once, then never again)
Look for `<root>/_tools/edit-config.json`. If it does not exist, ask the creator these questions in ONE message (multiple choice where possible), then save the answers there. Never ask them again; read the file at the start of every edit. If the creator later changes a style choice, update the file.

```json
{
  "language": "en",              // spoken language of the videos
  "scribe_language_code": "eng", // ElevenLabs Scribe code (ISO 639-3, e.g. eng, kat, rus, deu)
  "reply_language": "same as the creator writes",
  "sequence": {"w": 1080, "h": 1920, "fps": 30},   // vertical short default
  "accent_color": "#2B59FF",     // captions box, cards, rings
  "caption_style_name": "My Captions",             // Premiere Track Style name, created on first edit
  "caption_max_words": 3,
  "warmth": "auto",              // auto = measured per video (section 4), 0 = no colour work
  "brand_pack": false,           // true if the creator has an intro logo / outro / CTA (section 9)
  "logo_file": null,             // path under _library/images/ if brand_pack
  "cta_text": null,              // e.g. "Subscribe" or a website, shown in the outro
  "channel_handle": null,        // only if the creator wants it on screen
  "denoise_at_capture": false    // true if they already denoise while recording
}
```
If the creator does not want to answer, use these defaults and say so in one line.

### S2. Permissions (ask up front, explain why, one request per kind)
Ask for each permission ONCE, with a one-sentence reason, at the moment it is first needed in the session, and only for what the task needs. If a request is declined, do not repeat it: explain what cannot be done and continue with what can.
1. FOLDERS: call `get_device_info` to see connected folders. If the workspace root is not connected, call `device_request_folder_access` for the workspace root only (reason: "to read your recordings and write edits, renders and captions into your editing folder"). Ask for extra folders (e.g. the screen-recorder output folder, or `~/.cache/huggingface` for the Whisper fallback) only when that step is actually needed. Never request the whole home folder or a drive.
2. DELETING: only at the cleanup step (12), call `device_request_delete_permission` for the workspace root (reason: "to remove temporary files this edit created"). Delete only files the session created. If declined, move scratch into `<root>/_to_delete/` and tell the creator.
3. COMPUTER USE for Premiere Pro and After Effects: needed for things scripts cannot do (caption track style, speed dialog, clearing modal dialogs, starting the MCP bridge). First `computer_list_apps` to find the installed names (e.g. "Adobe Premiere Pro 2026", "Adobe After Effects 2026"; versions differ per machine), then `computer_resolve_access` with those names, then `computer_request_access` with the resolved entries (pass `willHide` exactly as returned). Reason: "to control Premiere Pro and After Effects for steps their scripting cannot do". Request it early in the session (access expires after ~30 min idle) and do all UI steps in one block.
4. FULL CONTROL: request `computer_request_full_control` only when a step needs display-level input (right-clicks, dropdown menus, typing into Properties fields). Before such a block, tell the creator in one line: "I'll control the mouse/keyboard for about a minute, please don't touch them." Release control when done.
5. BROWSER: only if a step needs it (e.g. downloading recordings from the creator's cloud photo library, or capturing a product page). Use the browser the session provides; never enter passwords or change account settings.
6. API KEYS: the creator stores keys in files under `<root>/_tools/` (e.g. `_tools/transcribe/elevenlabs.key`). Never print, log, commit to another place, show on screen, or put a key into any video, card or report. If a key is missing, ask the creator to create the file themselves.
Ask a question about the EDIT only for a decision that cannot be undone and could go either way (e.g. a screen recording that must be re-recorded).

### S3. Bootstrap the workspace (only what is missing)
Requirements on the creator's computer: Adobe Premiere Pro + After Effects, the Premiere Pro MCP bridge (CEP panel "MCP Bridge", started with "Start Bridge"), ffmpeg/ffprobe, an ElevenLabs API key for transcription (or the Whisper fallback). Remotion renders run in the cloud container (Node + Chromium).
If `_tools/` or `_library/` pieces are missing, create them yourself (write the scripts, keep a Mac `.command` and Windows `.bat` twin for every launcher):
- `_tools/transcribe/run_scribe.command|.bat(+.ps1)`: reads `_tools/jobs/job.env` (AUDIO, OUT), POSTs the audio to `https://api.elevenlabs.io/v1/speech-to-text` with `model_id=scribe_v1`, `language_code` from config, word timestamps, key read from `elevenlabs.key`; writes OUT; writes `exit=<code>` to `_tools/jobs/done.txt`; errors to `_tools/jobs/log.txt`.
- `_tools/captions/make_captions.py`: words + kept ranges + fixes -> SRT in timeline time (rules in 6.1).
- `_tools/ae/run_ae.command|.bat`: runs the jsx named in `_tools/ae/job.txt` in After Effects (Mac: `osascript` DoScriptFile; Windows: `AfterFX.exe -r`), writes `done` to `_tools/ae/done.txt`.
- `_tools/sfx/make_sfx.py`: synthesises short SFX with numpy (whoosh, pop, sting, hit, tick, keyboard typing; 48 kHz stereo wav) into `_library/sfx/`.
- `_tools/presets/`: copy the Premiere export presets you use ("Match Source - Adaptive High Bitrate", a low-bitrate preview preset) so paths work on any machine.
- `_tools/relink_root.jsx`: relinks offline items to the same relative path under the current root (after the folder is copied to another computer).
- `_library/images/white_backing.png` (white, sequence width x half height) for split-screen backing.
Check each new script once on a small input before using it in an edit.

---

## STANDING MODE: decide yourself, never ask
Do all of this every time without being told:
1. Look at EVERY file in the video's source folder (demo recordings, screenshots, images). Inspect them (contact sheet, `freezedetect`, a full-res frame) and decide whether and where they belong (split screen, b-roll).
2. Think like an editor: read the transcript, find the important moments (numbers, rules of thumb, waiting times, key terms, the core idea) and put a text/number/animation there. Compute what the speaker only says ("24 hours, how many minutes?" -> show `24 x 60 = 1440 min`). List them in the final report; no approval step.
3. Use your own creativity for visuals (section 13). Never leave a static screen for long. Cadence: a motion-graphics card every ~5-10 s (17.3), denser in the first 15 s (rule 8).
4. Remove retakes / false starts / dead air (2A).
5. Organize files and Premiere bins (workspace layout) and give every sequence a descriptive name (rule 11).
6. Screen hygiene (8.7): for every screen recording remove black strips, empty margins, irrelevant UI and PRIVATE info by crop or blur; verify on exported frames.
7. Typing on screen gets a typing animation + keyboard SFX. Zoom in where the action is on every screen recording (8.7).
8. HOOK: the first ~15 s must never be a plain talking head: zoom-out at the start (9), and from about second 4-5 a purpose-made explainer/animation (8.8) that illustrates what is said, so something new appears every ~4-6 s until the demo starts. One idea at a time, readable at phone size.
9. SPOKEN MISTAKES: when the creator says they misspoke (said 6:00, meant 9:00), correct the captions and every card to the intended value, keep the audio, and mention that the audio still contains the slip.
10. END OF VIDEO: no silent tail. Cut V1/A1 ~0.3-0.35 s after the last word; an outro/CTA animation plays OVER the last words and ends exactly at the end of the video.
11. NAMING: sequences get descriptive names, never raw clip ids: `NN <topic> (<variant>)` (e.g. `03 pricing-tips (final)`, `03 pricing-tips (short ~60s)`); topic from the transcript. Rename by `sequenceID` (`s.name = ...`). Exports use the same names.
12. FILLER REVIEW (12.7): list removable parts with timecodes; never cut them on your own and never auto-make shorter versions. The full version is the default deliverable.
13. FACE CENTRED horizontally on every video: measure per camera file (17.1), never reuse a fixed x.
14. FACE-ONLY WHEN THE SCREEN DOES NOT MATCH: when the speaker talks about something the screen recording does not show, switch to full face and carry the explanation with motion graphics (17.2).
15. NO DEAD AIR: after every cut list pauses > 0.8 s and remove the ones > ~1 s unless something important happens on screen.
16. REMOTION + MOTION GRAPHICS TOGETHER: when the speaker EXPLAINS a concept, process, flow, comparison or steps and no matching screen recording exists, build a Remotion scene that shows it (18), top half with the face below or full screen. Use AE motion-graphics cards (17.3) for short phrases, numbers and single facts. Never put a card on top of a Remotion scene.
The phone preview (16) is NOT automatic: at the end offer it in one short line and build it only on yes.

## RULE #0: one first, approval, then the rest
When the work produces several outputs (shorts cut from a long video, a batch of clips, variants), never build or export them all at once:
1. Analyse everything and put the plan (outputs with times, titles, priority, what extra recording would help) in a doc/canvas artifact.
2. Without waiting, make ONLY THE FIRST one, in Premiere, and say it's done.
3. The creator checks it (in Premiere, or asks for a preview).
4. After approval, do the next 2-3, report, wait again.
A new layout, style or pipeline always counts as unapproved: show one first.

## RULE #1: never undo or overwrite the creator's own changes
The creator edits by hand between turns (scale, blur, speed, trims, positions, deleted tracks...).
1. Before ANY change to existing clips, snapshot via ExtendScript every clip on every track (name, start-end, `getSpeed()`, in/out, components, Motion Scale/Position, Crop, effect values, keyframe counts). Any difference from what you last set = a creator edit.
2. Creator edits win. Change ONLY the requested property in place (`setValue`). Select clips BY NAME, never by index. Add new things on free ranges/tracks instead of rebuilding.
3. If a rebuild is unavoidable, record all effects/values, rebuild, re-apply every one, or ask first. Say in one line what was kept.
4. Re-read the project right before each batch of edits.
5. A change you did not make is the new baseline.

## Workspace layout (must stay copyable between Mac and Windows)
ROOT = the folder that contains `_tools`. In Premiere ExtendScript derive it from the open project (`new File(app.project.path).parent...`), never hard-code `/Users/...`, `C:\...` or `/tmp/...`. In the device shell the root appears under `$HOME/mnt/<folder>`: list `$HOME/mnt/` first.
```
<root>/
  _tools/        scripts only (launchers, presets, jsx, python), edit-config.json, jobs/ (scratch: job.env, done.txt, log.txt)
  _library/      shared assets: animations/brand, animations/keyphrases, sfx/, images/
  _backup/
  <YYYY-MM-DD>/                 one folder per recording day
    <NN topic>/                 one folder per video
      source/                   the creator's media + canonical SRT + this video's own renders (animations/, remotion/)
    _work/<NN topic>/           archive: transcript JSON, words/ranges/fixes, old SRTs, scripts used
    _work/previews/             preview exports
    export/                     final exports (superseded ones in export/_old/)
```
- ONE PREMIERE PROJECT PER LONG VIDEO (big projects freeze Premiere): `<day>/<NN topic>/<NN topic>.prproj`, containing the video's bin and a `Shared` bin.
- Never write scripts, JSON, logs, frames or exports into `source/`. Never create derived/re-encoded copies of the creator's media (use Premiere's speed/crop/scale). Never delete or move the creator's files.
- Shell work (mkdir/cp/ffmpeg/python) via `device_bash` in the connected root. Things that must run on the computer itself (AE, transcription launchers) run via ExtendScript `new File(path).execute()`. Files made in the cloud container reach the computer with `device_commit_files` (see 14.1 size limits).
- OS check in ExtendScript: `var WIN=$.os.indexOf("Windows")>=0`; run the matching launcher twin. `.bat`/`.ps1` use CRLF; after editing a `.command` run `chmod +x`.
- PREMIERE BINS mirror the folders: one bin per video holding ALL of its items (every sequence, source media, SRTs, its animations); `Shared` holds only shared items (intro/outro, sfx, caption style). No loose root items. `rootItem.createBin(name)`; move with `item.moveBin(bin)` (collect items into an array first, then move). Before importing, reuse an existing item with the same media path. Save at the end.

## 0. Preflight
1. `verify_premiere_connection` (or `ping`). If it fails: open Premiere, open the MCP Bridge (CEP) panel, check its temp directory (Mac `/tmp/premiere-mcp-bridge`, Windows `%TEMP%\premiere-mcp-bridge`), click Start Bridge (needed after every Premiere restart and after the last project was closed).
2. Find the video's sequence (`get_active_sequence`, or list sequences). If none exists: import the clip and create it with `app.project.createNewSequenceFromClips(name,[item],bin)` then `set_active_sequence`. If the source is 4K, edit in the config sequence size (1080x1920 @30) unless the creator wants otherwise. NEVER `createNewSequence(name,id)`: it opens a dialog and hangs ExtendScript. If calls time out (~45 s), take a screenshot: a modal dialog is blocking; cancel it. Clip nodeIds change on razor/re-place/speed: re-read right before use. Take the RULE #1 snapshot.
3. ExtendScript: wrap in `(function(){...})()` and `return` a string. NEVER write a loop that may not terminate (hangs Premiere). Add tracks with `add_tracks` or QE `addTracks`. Numeric tool params must not carry trailing zeros (`90.1`, not `90.10`).
4. Permissions per S2 (folder, Premiere + AE computer access) before the first step that needs them.
5. Computer-use notes: the `computer_app_*` background tools often refuse right-clicks and do not show dropdowns; display-scope tools need full control. Bring an app to the front with `computer_open_application`. A window on another Space/desktop needs `computer_app_bring_to_current_space`. Call `computer_app_release` before switching back to display-scope tools. Clicks are ignored while the creator uses the mouse (playhead jumping in screenshots): ask them to keep hands off for a minute. With several monitors check `computer_cursor_position`: if clicks land on another monitor, use keyboard and background clicks. Always-on-top overlay apps (webcam/ring-light helpers, recording widgets) can block clicks. Browsers are read-only for computer use. Screen recorders (OBS etc.) cannot be driven: the creator records demos. OS Open/Save panels cannot be driven: use `open_project`/`import_media` tools.
6. Never type into the timeline with no focused field: letters are shortcuts (I/O set in/out, Return plays). If it happens: `clear_sequence_in_out` / Space.

## 1. Analyse audio (ffmpeg in the device shell, `-vn`)
- Per channel `-af "pan=mono|c0=c0,volumedetect"` and `c0=c1`. One channel < -60 dB mean and the other > -40 dB -> one-sided audio: `Fill Right with Left` (or Left with Right) via `apply_audio_effect_to_all_clips`.
- Loudness `-af ebur128=peak=true -f null -`: integrated LUFS and true peak. Filling the empty channel adds ~+3 dB.
- Noise floor of silent parts (`silencedetect` + `volumedetect`): decides DeNoise (3).
- Camera file + screen recording of the same take: read section 15 first.
- 4K HEVC phone footage decodes slowly in the device shell: sample frames in parallel chunks (`-ss C -t 20`) or with `-skip_frame nokey`, never one whole-file fps filter.

## 2. Cuts
### 2A. Retakes, false starts, dead air (from the transcript, no instruction needed)
For each group of takes with the same content keep ONLY the most valuable one (almost always the LAST complete take); cut the others, including dangling half-sentences and restarts. Method: transcribe (5) BEFORE cutting, print the transcript grouped by pauses > 0.5 s with word timestamps, mark keep/drop, put cut points inside silences (`silencedetect` noise -35 dB d 0.4; start+0.1 / end-0.1; never inside a word; 0.05-0.1 s padding after the last word). If the tail of a kept take repeats the start of the next, trim the tail. List dropped takes in the report and mention dropped content that carried unique information.
DEAD AIR: waiting with nothing happening (e.g. a page loading) is cut or covered with an animation. But an ACTION the viewer needs (typing the question the next sentence comments on, the answer appearing) stays, shortened.
### 2B. Build the cut timeline
PREFERRED (exact, no drift): empty the sequence and place the kept source ranges one after another: `item.setInPoint(t,4); item.setOutPoint(...); track.overwriteClip(item, at)` with `at` = previous clip end; then `item.clearInPoint(); item.clearOutPoint()`. Close 1-frame gaps by setting `clip.start`/`clip.end` to the neighbour's boundary (V1 and A1). Razor (`razor_timeline_at_time`) is the fallback: it lands early/late by an inconsistent amount and cuts all video/audio tracks (not captions): always read the real boundaries afterwards; if a boundary is inside a word, re-razor.
- Record the V1 MAPPING `[tlStart, tlEnd, srcStart]` per clip from the LIVE sequence and convert source -> timeline through it. NEVER reuse ranges/SRT/timecodes from an earlier version (sequences drift).
- END: V1/A1 end = last word end + ~0.3-0.35 s. An outro/CTA ends at the sequence end and plays over the last sentence, placed where it does not cover the face. If the creator trims it, keep their version.
- MOVING A CLIP: setting `clip.start` TRIMS the head and does not advance `inPoint`. To move: record inPoint and duration, set `start`, set `end = start + duration`, restore `inPoint`. Shift everything after a cut on every track by the same amount and re-map captions/animations.
### 2C. Cut variants and removing spans
1. "Remove a to b seconds" refers to the latest full version; if several versions exist and it matters, confirm which in one line. When the creator quotes words, find the phrase in the transcript (source times) and cut on word gaps; print the section's words with timestamps before cutting.
2. Make the variant as a NEW sequence (`sequence.clone()`; find the clone by comparing sequenceIDs before/after, never by index); never edit a version already reviewed (unless asked to fix THE working sequence in place). Never delete a variant the creator rejected.
3. SPAN REMOVAL: razor all tracks near both cut times (QE per-track razor after `setPlayerPosition`), read real boundaries, then ONE script per span, LATER SPAN FIRST: `D=Rb-Ra`; clips fully inside -> `remove(false,false)`; clips after -> move by -D (2B method). Close 1-frame gaps, verify V1 has no gaps/overlaps and ends at old end - sum(D). `sequence.end` is stale after edits: compute the end from the last clip.
4. Rebuild the SRT from words whose midpoint lies in kept ranges (live table), import, `create_caption_track`, style (7). A cloned sequence carries the OLD caption track: hide or delete it (6.3).

## 3. Noise reduction and level
- DeNoise per video: apply `DeNoise` on all audio clips when the silent-part noise floor is above ~-60 dBFS, unless `denoise_at_capture` is true or the creator says it is already clean. Measure before/after if unsure.
- Volume > Level is RAW linear: 0.1778 = 0 dB, 1.0 = +15 dB. Effective loudness = I (+3 if a channel was filled) + 20*log10(level/0.1778). Target about -14 to -16 LUFS for social video; |gain| <= 1.5 dB -> leave it; clamp [-12,+12] dB, true peak <= -1 dBTP; Level = 0.1778*10^(gain/20) via ExtendScript (component "Volume", property "Level", `setValue(v,true)`) on every audio clip, also clips created later by razor. Alternatively a Hard Limiter (max amplitude ~-0.1 dB) after the gain.
- SFX levels: start low (raw 0.012-0.07); creators usually find SFX too loud.

## 4. Colour (skip if warmth = 0)
`color_correct` takes a `clipId` (one call per V1 clip). PITFALL: Lumetri saturation 100 = normal; the tool's low saturation values turn the picture black-and-white: afterwards set every numeric "Saturation" property to 100-110 via ExtendScript and verify. Warmth is decided PER VIDEO: measure the face colour first (mean R/B on face crops), an already warm room needs little or even negative Temperature; compare a graded frame with the original, then set the value. Aim for a slightly warm, natural skin tone.

## 5. Transcript (from the audio; never ask for a script)
Primary: ElevenLabs Scribe (word timestamps, strong on many languages).
1. Device shell: `ffmpeg -i <video> -vn -ac 1 -b:a 64k <root>/_tools/jobs/audio.mp3` (one-sided audio: take the live channel).
2. Write `_tools/jobs/job.env` (`AUDIO="..."`, `OUT="..."`), `echo running > _tools/jobs/done.txt`.
3. Run the launcher via ExtendScript `new File(ROOT+"/_tools/transcribe/"+(WIN?"run_scribe.bat":"run_scribe.command")).execute()` (if the first run hangs, execute again). Poll `done.txt` for `exit=0` with sleep loops < 170 s per call (background processes in the device shell die between calls). If curl from the device shell works, you may call the API directly from there instead. Filter the JSON `words` list to `type == 'word'`.
4. Archive the raw JSON in `_work/<NN topic>/`. Fix only clear recognition errors with a fixes dict (product names, brand names, technical terms the model mis-hears; keep a per-workspace list in `_tools/captions/fixes_common.json` and extend it when the creator corrects a word). List remaining uncertain words in the final message instead of asking.
Fallback: Whisper large-v3 (faster-whisper) on the creator's computer via a launcher; the cloud device shell is too small for it.

## 6. Captions
1. Keep only words whose midpoint lies inside a kept source range. `make_captions.py words_kept.json ranges.json out.srt fixes.json <max_words>`: <= max_words per cue, balanced groups (8 words -> 3,3,2), sentence break on .?! or gap > 0.6 s, min cue 0.45 s, gaps < 0.35 s closed, no overlaps, output in TIMELINE time.
2. Canonical SRT in `source/` (the creator may edit it); revisions get new names (`_v2`; Premiere caches by path). After ANY trim of V1 rebuild the SRT via the live mapping.
3. `import_media` (one file per call), then `create_caption_track` (`projectItemId` AND `sequenceId`): it only creates an unstyled track. Deleting a caption track and styling are UI-only (right-click track header > Delete Track needs full control) or simply HIDE the old track with its eye icon (hidden caption tracks are not burned into exports). ExtendScript cannot read/edit caption tracks. Two caption tracks with different positions (seam captions for split parts, bottom captions for full-face parts) are made from two SRTs split by cue midpoint.
4. Spoken-mistake fix: correct the value in the SRT -> `_v2`, import, new track, hide the old one, style, verify a frame; re-render affected cards.
CAPTIONS FALLBACK (UI styling impossible): hide caption tracks and burn captions as a transparent Remotion overlay on the top video track (cues from the SRT, accent box, white bold text, safe-zone y). Premiere burns every VISIBLE caption track into exports.

## 7. Caption style (UI, Properties panel)
Do this FIRST among UI steps, while computer access is fresh, with the creator's hands off the mouse. If the Track Style `caption_style_name` exists in the project: click the first caption, shift-click the last, Properties > Track Style dropdown > the style (dropdown items need full control; use `computer_screenshot`, not app screenshots). If it does not exist yet (first video): edit ONE caption: bold font that supports the language's script, size ~54 at 1080 width (scale with sequence width), position in the lower safe zone, background = accent colour at ~90% opacity with rounded corners, white text; then create the Track Style from it ("+" next to Track Style) named `caption_style_name`, and apply to the whole track. Numeric fields: double-click, select all, type, Return. Verify on 3+ frames incl. one over bright skin, and click a MIDDLE caption to confirm the style changed. If clicks do not register, use the CAPTIONS FALLBACK or tell the creator once. Never loop on failing clicks.

## 8. Overlays / split screen
RULE #1 first: if overlay tracks exist, adjust only what was asked, in place.
### 8.1 Recordings
- Inspect every recording in `source/` and decide placement yourself. Good demo recordings start from a clean/empty state, have the app zoomed so text is readable on a phone, and show only the relevant pane; if a recording breaks these badly, say so (ask only if a re-record is needed).
- When the speaker names a setting, show the SETTING screen first for a few seconds, then the demo.
- CAMERA TURNED TO THE SCREEN: when the camera itself films the computer screen (detect with a face detector over time + contact sheet), that real footage IS the demo: keep it full screen, never cover it with a recreation; only small chips on top.
### 8.2 Moments, source range and speed (computed per video)
- Place overlays at the timeline times where the topic is spoken; match by meaning. A static page may stay only while it is the subject; when speech moves away, go face-only (17.2).
- Detect the action: crop to the region of interest, `freezedetect=n=0.003:d=0.4` and `select='gt(scene,0.002)',showinfo`; confirm with a contact sheet (`fps=1/2,scale=320:-1,tile=4x3`, stage and Read). actionStart = first change that matters; actionEnd = last meaningful change. Blinking cursors/idle dots are not action.
- ALIGN WITHOUT SPEED CHANGES where possible: choose the in-point so the key action lands when the speaker says it (the result may lag the word, never precede it). Otherwise speed = actionLen / (slotsTotal - hold), hold ~1.5 s; <= 1.05 -> 100%. Never reuse a speed from another video.
### 8.3 Layout (vertical, overlay share s = 0.4 by default)
Overlay zone = top s*H; the SEAM is y = s*H.
- V2: white backing over the overlay interval, cropped to the zone.
- V3: overlay clips cropped to the useful area and scaled to COVER the zone (text readable at phone size: crop tight rather than shrinking). The Motion component has built-in `Crop Left/Top/Right/Bottom` (percent) in recent Premiere versions; Scale 100 = native pixels. Position (px) = desiredCentre - (visibleCentre - sourceCentre)*scale, then normalise by W/H.
- To change a crop while keeping the visible rect: scale' = scale*(1-T-B)/(1-T-B').
- V1 speaker under the overlay moved down so the head top is ~5% of H below the seam; set position on the V1 clips inside the interval (razor at the interval edges).
- Captions in split layouts sit ON THE SEAM, never over the face.
### 8.4 Placement mechanics (ExtendScript)
- Video: `item.setInPoint(sec,4)`, `item.setOutPoint(sec,4)`, `track.overwriteClip(item, at)`; trim with `clip.end`. Overwrite uses the item's WHOLE in/out span: never overwrite a span longer than the slot on a populated track. After placing, set `clip.inPoint` exactly. Re-list the track after every step. `numClips` is unreliable: iterate with try/catch.
- Avoid `move_clip_to_track` (overwrites neighbours, ignores speed).
- Overlay audio lands on A3 (mute it unless it is the chosen voice, 15); SFX on A2/A4, non-overlapping per track.
- Motion: `clip.components[1]`, match properties by `displayName` (`Scale`, `Position` normalised 0-1, `Crop *`). Keyframes: time = inPoint + (timeline - start); `setTimeVarying(true)`, `addKey`, `setValueAtKey(t,v,true)`; hold keys `setInterpolationTypeAtKey(t,4,true)`. Never QE `addVideoEffect` for Motion work. Split/extended clips keep all keys of the original: remove ones that do not belong.
- Effects cannot be removed by script (UI only).
### 8.5 Speed (UI only)
`speed_change`/QE `setSpeed` do not work. Select the single clip (`setSelected(true,true)`), Premiere frontmost, timeline focused, Clip > Speed/Duration via `computer_app_menu`, then type the value (select all, type, Tab, Return). Check the resulting duration; extend `clip.end` to the slot end if needed.
### 8.6 Verify
`export_frame`/`capture_frame` ignore the time: do not use. Use QE `qe.project.getActiveSequence().exportFramePNG(timecode, pathWithoutExt)` after setting the player position (never overwrites: new prefixes), or export the sequence and grab frames with ffmpeg (more reliable for timing). Tile frames into a contact sheet and Read it: start/middle/end of every overlay, every zoom pose, intro/outro, every cut seam.
### 8.7 Screen hygiene, zoom, privacy blur (every screen recording)
- BLACK BARS / MARGINS: full-res frame, row/column means with numpy; runs with mean < ~60 are bars: crop at least 10 px inside; also crop irrelevant browser chrome/sidebars; re-fit scale to cover the zone; verify no bar remains.
- ZOOM ON ACTION: keyframe Scale+Position on the screen clip: P0 wide, P1 tight (~1.9x) on the control a bit BEFORE the click, P2 medium containing control + result, back to P0 after. Transitions 0.5-0.6 s, holds >= 1.5 s. Do not zoom so far that context the viewer needs (which app/tab is acting) disappears. Never animate a zoom-out below the fitted size.
- PRIVACY (never ask, always do): nothing private may be readable on any frame, including the first and last frame of a blur: email addresses, phone numbers, API keys/tokens, passwords, private system prompts or configuration, other people's names/chats, customer data, browser tabs/bookmarks with private titles, private apps (DB tools, banking). If the creator says certain items may be shown, follow that. Methods: Gaussian Blur (Legacy) on a copy cropped to the region, or "blur everything except region" (blur the base clip, put a sharp cropped copy above). Detect regions with numpy on sampled frames; add margins. A blur copy over a keyframed screen clip must carry EXACTLY the same Position/Scale keys. Verify at every zoom pose and at both ends of every blur.
- NEVER use the GPU "Gaussian Blur" effect (shows "Error occurred", cannot be removed). Only "Gaussian Blur (Legacy)".
### 8.8 HOOK: the first 15 seconds
- 0-~3.5 s: zoom-out on the face + whoosh (9), optional brand intro. From ~4-5 s: a purpose-made explainer (Remotion or AE) in the overlay zone that illustrates what the speaker says (who is involved, what they do, the key idea), timed to word timestamps, ending exactly where the demo starts. Something new every ~3-5 s, pop-in on the spoken word (+0.0-0.2 s), SFX, max 2-3 elements moving at once.
- Wording on every visual must be factually right for the creator's subject: describe who really does what; if unsure about a business fact, keep the wording neutral and flag it in the report.

## 9. Brand pack, zoom-out, sound effects
- Brand pack (only if `brand_pack` is true): intro logo animation at 0 s, an outro/CTA animation that ends at the sequence end. Render once with AE or Remotion from `logo_file`/`cta_text` into `_library/animations/brand/` and reuse. Place them where they never cover the face (scale them down, e.g. intro centred in the lower third, outro above or below the face depending on framing); check a frame.
- ZOOM-OUT on the first V1 clip (fast and far): Scale keys 0=base*1.75, 0.1=1.38, 0.2=1.12, 0.3=1.03, 0.4=1.008, 0.5=base. Remove old keys with `removeKey`.
- SFX are synthesised (`make_sfx.py`): whoosh at 0 s, sting on the logo, pop on every card, tick for counting, keyboard for typing. EVERY animation/overlay that pops up gets an SFX. Raw Level 0.03-0.07; short, simple, not busy; keep the creator's levels.

## 10. After Effects pipeline
1. Write the jsx in `_tools/ae/` with a head that defines `ROOT` (`(new File($.fileName)).parent.parent.parent.fsName`), a script-aware font variable (see fonts below) and helpers (`shapeLayer`, `addText`, `ring`, `burst`, `card`, `queue(comp,name)`, pop-scale). Several comps can be queued in ONE script. Write the job name to `job.txt`, `echo running > done.txt`. A jsx written in the cloud container reaches the computer via `device_commit_files`.
2. Run via ExtendScript `new File(ROOT+"/_tools/ae/"+(WIN?"run_ae.bat":"run_ae.command")).execute()`. Poll `done.txt` and `ae/out/log.txt` (clear it before a run) with sleep loops < 170 s.
3. Stuck on `running` = AE shows a dialog (script error, "Save changes?", "Overwrite?"): bring AE to front, press Return/Escape. Avoid by starting with `app.project.close(CloseOptions.DO_NOT_SAVE_CHANGES)` and a NEW output name per iteration. One-time AE setting: Scripting & Expressions > Allow Scripts to Write Files and Access Network (tell the creator if it is off).
4. Facts: render template "High Quality with Alpha" (ProRes 4444); text animators `ADBE Text Animator` + `ADBE Text Selector`; overshoot via expression on Scale; trim paths `ADBE Vector Filter - Trim`; parenting does NOT carry opacity; Gaussian Blur on a shape layer is clipped to layer bounds. FONTS: pick a font installed on BOTH Mac and Windows or branch by OS; non-Latin scripts (e.g. Georgian, Armenian, Arabic) need a font that contains them (check on a test render); some currency signs and EMOJI do not render in AE: write the currency as a word and draw icons from shapes.
5. Check every render: `ffprobe` and a contact sheet over grey. Re-render with a NEW name when anything is wrong; move renders out of `_tools/ae/out/` into `source/` (video-specific) or `_library/` (shared).

## 11. Key-phrase animations (autonomous)
1. CADENCE: pick important moments (numbers, durations, comparisons, rules of thumb, calculations, key terms) so a new card appears every ~5-10 s (every 4-6 s in the hook). Solve computations the speaker poses. Each card 2.4-8 s. List them in the report.
2. LISTS/STEPS: rows LEFT-ALIGNED on one vertical line with number badges; each row pops in when its item is spoken (current SRT times).
3. TYPING: text typed on screen (or a loading wait) gets a typing card with a caret + keyboard SFX over exactly the typing span.
4. Visual language: accent-colour rounded card, white stroke, drop shadow, pop in/out, rings/confetti sparingly, big white text, vector icons. Keep templates in `_tools/ae/` and reuse them.
5. Placement: centred on the seam in split layouts; with no overlay, where the face is not (above the head or on the chest area); check a frame. No collisions with intro/outro or each other.
6. SFX per card on A2 at the same start, trimmed to the card end.
7. Verify mid-animation frames: face not covered, captions legible, text aligned.

## 12. Finish, organize, cleanup, report
1. Verify via ExtendScript: effects per clip, caption count = SRT cues, track listing = plan, nothing the creator changed reverted, no offline media.
2. SEQUENCE END = outro end = last word + ~0.35 s; no trailing silence; clear stray in/out points.
3. Organize per the workspace layout; relink moved items with `changeMediaPath(newPath,true)`; descriptive names; no duplicate items.
4. Cleanup: archive transcript/words/ranges/fixes/old SRTs/scripts in `_work/<NN topic>/`; request delete permission (S2.2) and delete only session scratch (`_tools/jobs` temp files, `_tools/ae/out/*`, failed renders); never delete the creator's media, Premiere Auto-Save or `export/`.
5. Save the project. Do not export unless asked.
6. Report 3-6 short lines: cuts (old -> new duration, dropped takes), audio (gain, DeNoise yes/no and why), colour, captions, overlays/zooms/speed, key-phrase animations (timecodes), privacy blurs, corrected slips, uncertain words, anything MISSING (assets, styles, logos, recordings) stated explicitly. Mention that you cannot hear audio/SFX and ask the creator to check. End with the one-line preview offer.
7. FILLER REVIEW: when shorter versions are wanted, list removable segments as `[m:ss-m:ss] "first words" - why` with resulting durations; build the chosen one as a NEW sequence. Never cut the sentence that states what the video is about, or nuance sentences that carry the main message. When the creator says "I repeat myself, remove what's redundant", decide and cut yourself, then list what was removed.

## 13. Creative visuals: never static, never boring
Scan the finished timeline: any stretch > ~10 s without an animation (> ~5 s in the first 15 s), > ~6-8 s of a static screenshot, or the speaker alone while a concept is explained, gets a purpose-made visual you design without asking.
- First choice for explanations > ~6 s: a Remotion scene (18). Also good: an AE explainer built from shapes, beat by beat on word timestamps (bubbles, typing dots, avatars, timers, switches, counters, flows a -> b -> c).
- Place over the overlay zone (or full screen), ending exactly at the next cut; the static recording stays underneath. SFX at the beats, low level.
- Other options: kinetic text, a counter/calculation card, a transition (whoosh + slide), occasional zoom punch on the face. Keep face zoom changes infrequent (too many are tiring). Simple, clean, readable at phone size.
- Verify 3+ frames: nothing cut off, nothing overlapping, captions legible.

## 14. Pitfalls and tool limits
- Audio Level 0.1778 = 0 dB.
- Razor lands with an inconsistent offset: read real boundaries, re-razor if inside a word. QE per-track razor cuts only chosen tracks.
- `overwriteClip` uses the item's whole in/out span; overwriting on an audio track also puts video on the same-index video track and can cut earlier SFX. `sequence.insertClip` ripples only target tracks: use the UI insert edit when everything must ripple.
- `clip.start =` trims the head (2B). `remove(false,false)` removes without ripple (video item only; linked audio stays).
- Frame export: only QE `exportFramePNG` honours time.
- ExtendScript cannot remove effects, edit caption tracks or run shell commands: use launchers via `new File(...).execute()`. The `execute_extendscript` validator rejects `eval`/`new Function`: put long code in a .jsx and `$.evalFile` it. It returns a value only with an explicit `return`.
- `apply_audio_effect*` needs the clip id from the ACTIVE sequence.
- `export_sequence` may falsely report Media Encoder missing: use `app.encoder` (16).
- `extract_selection`/`ripple_delete` act on SELECTED clips, not the in/out range: never use them to remove a time span.
- A stuck modal dialog makes every ExtendScript call time out: screenshot, find it, cancel it. A "Link Media" dialog on project open: Escape, then relink by script.
- SEVERAL PROJECTS OPEN: `app.project` is not necessarily the one opened last; address `app.projects[i]` explicitly (check `.path`) before destructive work. Closing the LAST open project unloads the MCP Bridge panel.
- If another chat is using Premiere at the same time: never switch or close its project; open yours with `app.openDocument(path,true,true,true,true)` and edit through `app.projects[i]`; export long videos via `app.encoder.encodeSequence` + `startBatch()` so the scripting host is not blocked.
- ExtendScript hang recovery: save, quit via the menu, reopen the project, press Start Bridge; tell the creator honestly.
### 14.1 Speech cuts and file delivery
- Never chop words. After any cut that touches speech, transcribe the exported range to verify. If a sentence still sounds damaged after two tries, put back the original continuous clip for that sentence.
- `device_commit_files` rejects files above ~26 MB and may silently NOT overwrite an existing file: always commit under a NEW name and relink with `changeMediaPath`. It also inserts a metadata box into MP4s: split big files into parts < 20 MB and join them with plain `cat` on the device without stripping the box; check with an ffmpeg decode. Never overwrite a media file Premiere already uses.
- ffmpeg input `-ss` + `fps` keeps pre-roll frames and drifts sync in concatenations: use `trim=start=0:duration=d,setpts=PTS-STARTPTS` (+ `atrim/asetpts`) before `fps`.
- Device shell jobs die when the call ends: keep each job under ~170 s per call.

## 15. Two recordings of the same take (camera + screen)
Only when a camera file and a screen recording cover the SAME speech. A separate demo recording is not this case. Decide each from evidence:
1. SYNC: needed when the screen recording's picture/audio is used along with the camera: cross-correlate both audio tracks (mono 16 kHz, numpy FFT) for the offset, confirm with contact sheets. Not needed when one file supplies both picture and audio.
2. WHICH AUDIO: compare loudness and noise floor; use the cleaner one (screen recorders with a good mic are often cleaner); if equal keep the camera audio. If the screen audio is the voice it lands on A3 unmuted and A1 is removed.
3. DENOISE: per section 3.
When synced: transcribe that audio (one time base), cuts in camera time, screen in-point = camera in-point + offset; keep A3 (inPoint - start) equal to V3 (inPoint - start) on every clip. Build order: sequence from clips -> remove clips -> place V1 pieces -> close gaps -> place V3 pieces -> remove A1 -> fix in-points -> tracks -> colour -> layout -> captions -> style -> bin -> save.

## 16. Preview and export
The creator often reviews on a phone, but not always: offer a preview in one line; build it only on yes (or when asked).
1. Export via ExtendScript: `app.encoder.launchEncoder(); app.encoder.encodeSequence(seq, out, ROOT+'/_tools/presets/<low bitrate preset>.epr', app.encoder.ENCODE_IN_TO_OUT, 1); app.encoder.startBatch();` with in/out = 0 -> last V1 clip end (the sequence end can be longer and adds a black tail). Output folder must exist; previews only into `_work/previews/`.
2. Poll until the size stops growing and `ffprobe` shows the expected duration. Media Encoder writes temp files first: the export is done when the final `.mp4` exists and no temp remains.
3. Shrink: `ffmpeg -i master.mp4 -vf scale=540:-2 -c:v libx264 -crf 28 -preset veryfast -c:a aac -b:a 96k -movflags +faststart preview.mp4`.
4. Deliver: `device_stage_files`, then `SendUserFile` with `display:"render"` so it plays inline (no download needed).
5. FINAL EXPORT (only when asked): preset "Match Source - Adaptive High Bitrate", into `<day>/export/<NN topic>.mp4`; new versions get `(vN)`; move superseded exports to `export/_old/`. Check duration and resolution with ffprobe.

## 17. Face framing, face-only sections, motion-graphics density, revisions
### 17.1 Centre the face (every video)
- Measure: `pip install --user opencv-python-headless` in the device shell if needed (some cv2 builds lack CascadeClassifier), sample ~14 frames, Haar `haarcascade_frontalface_default`, median face centre (faceX, faceY as fractions of the source).
- With k = rendered width / sequence width (Scale/100 x source width / sequence width): x = 0.5 + (0.5 - faceX)*k; clamp x so no black edge appears (scale up if needed: Scale >= base*(1 + 2*|x-0.5|)). Set x on EVERY V1 clip, keep each clip's y and keys. Verify on exported frames: face centre 0.47-0.53 of the width.
### 17.2 Face-only sections
- Cut at the silence before the first word of the section; razor V1/V2/V3 (and blur tracks); right part: V1 full-face framing; remove the overlay pieces with `trackItem.remove(false,false)` (voice on A3 stays; check its clip count).
- In full-face frames put cards on the free area (chest or above the head), never on the face; captions at the bottom safe zone.
### 17.3 Motion-graphics density and design
- Cadence ~5-10 s, each card 2.5-8 s, pop on the word (card start ~0.3 s before). Never over a full-screen screen layout where the card would hide text being read.
- Content: 2-5 words + a vector icon (file icons, check, X, warning, chips, avatars, arrows/flows, stars, mic, faces, counters). Never real private data (masked digits only).
- One AE script can build all cards of a video. Check renders on a contact sheet (text overlapping icons, chips off the card, invisible colours); re-render only broken comps with a new suffix.
- Store in `source/animations/`; import with `app.project.importFiles(paths,true,bin,false)`; place on an empty track above the overlays; SFX on an empty audio track.
### 17.4 Revision workflow
- Re-snapshot every sequence first. "Review but don't change" = report only. "Change nothing else" = only the requested parts.
- Creator timecodes are Premiere display times; locate exact words with the transcript + an RMS energy scan (20 ms windows); cut in silence.
- REMOVE A SPAN in place: set sequence in/out, target all tracks incl. captions, focus the timeline, `computer_app_menu ["Sequence","Extract"]` (ripples captions correctly). Then close 1-2 frame gaps. If a caption still shows removed words, edit its text in the Properties panel (needs full control).
- RE-INSERT A MISSING ACTION: open the item in the Source monitor with in/out, set sequence in/out to the slot, patch source tracks, Insert (`,`), then fix the gap on the sync-locked camera track and copy Motion/Lumetri from a neighbour.

## 18. Remotion explainer clips
### 18.1 When
- Remotion = a SCENE showing what the speaker explains, beat by beat: architecture/flow diagrams with moving packets, browser walkthroughs with a typing URL and cursor clicks, message flows, comparison columns, timelines, step-by-step processes. Use it whenever an explanation is not on a screen recording and lasts > ~6 s.
- AE cards (17.3) = short facts, numbers, single phrases, < ~5 s. Mix both per video.
- Concepts get ANIMATED diagrams, not static lists (e.g. integrations as a hub with spokes and pulses, version history as a timeline with a rewind, sync as packets travelling between devices and a cloud).
### 18.2 Layout
- Split: panel on the top part (V3, opaque H.264 at real size, Scale 100, Position (0.5, panelH/2/H)), face in the bottom part (move V1 y with 2 keys 1 frame apart at the split edges). Hook and outro stay full face. Full-screen scenes are allowed for dense content (face hidden).
- Full-face with TRANSPARENT overlays: compositions at sequence size with no background (title pill above the head, chips on the chest, never over the face), rendered ProRes 4444 then converted to `qtrle` argb in <= 10 s chunks so each file stays < ~25 MB.
- Scenes contiguous (a 1-frame gap shows black): `prev.end = next.start` by ticks. Whoosh at every scene start. Remove MG cards inside Remotion spans.
### 18.3 Project (renders in the cloud container; the computer only receives videos)
- Keep the project archived in `_tools/remotion/` (helpers, scenes, Root, public/fonts, logos). Copy into the container, `npm i` (remotion@4, @remotion/cli, @remotion/renderer, react@18, react-dom@18, typescript, @types/react, fonts that cover the video language, e.g. `@fontsource/noto-sans-<script>` + `@fontsource/inter`).
- Fonts via `@font-face` + `staticFile()`; `Noto Color Emoji` renders emoji in headless Chromium.
- Render: `npx remotion render src/index.ts <Comp> out/<name>.mp4 --codec=h264 --crf=18 --concurrency=2` (use the headless Chromium the container provides). Check STILLS first (`npx remotion still ... --frame=N`): text wrapping, elements off canvas, overlaps, labels on top of lines.
- Design: the creator's accent colour, dark or light background with subtle motion, white cards, green check / red X, spring pops, typing text with a cursor, arrows, travelling bubbles, cursor with click ring. One idea at a time, readable at phone size.
- Timing: every element keyed to absolute timeline seconds taken from the word timestamps; for a shortened version map original -> new times through the cut list and keep interpolate ranges strictly increasing.
- OFFICIAL LOGOS wherever a product/service is named (never emoji for brands). Sources: `@iconify-json/logos` (npm), `@lobehub/icons-static-svg` (npm, AI brands), `simple-icons` (npm, monochrome), `raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/<name>.svg`. Dark logos need a white circle behind them. Tell the creator where each logo came from.
- Delivery: `device_commit_files` into `source/remotion/` under NEW names, import into the video bin's `remotion` sub-bin.
### 18.4 Shorter versions of a finished sequence
`sequence.clone()` copies the caption track, which cannot be shifted by script: when timing changes and real caption tracks are used, REBUILD instead: compute kept pieces from the V1 clips + cut list, create a new sequence from the camera item, place pieces, copy every component of the original clip index-wise, add missing effects via QE on the item found by START TIME, copy brand items, shift the outro to the new end, close gaps, new SRT from kept words, new caption track, layout keys. Verify every V1 clip has the right colour, audio level and effects, 0 gaps. Find clones by sequenceID, never by index.
### 18.5 Portrait phone clips
Premiere shows them rotated to portrait; audio is often left-channel only (Fill Right with Left); frame rate may be reported oddly. Centring the face by moving x leaves a black strip at Scale 100: scale up so the frame still covers.
