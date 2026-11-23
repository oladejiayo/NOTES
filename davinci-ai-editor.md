

# Multi-cam Sync
https://brianrossen.com/blog/multicam-sync/

Give Claude Code this repo's link and say something like:

- Set up resolve-multicam-sync from and sync everything in "/path/to/Shoot Day" into Resolve.
- https://github.com/brianrossen/resolve-multicam-sync

The easiest way is with Claude Code, so you don't have to touch the setup yourself. I used it with Claude Opus 5.5 as the model.

- Open Resolve Studio with a project.
- In Claude Code, say: "Set up https://github.com/brianrossen/resolve-multicam-sync and sync everything in my Shoot Day folder into Resolve."
- Claude installs what it needs and does a dry run first. It reads you the summary: each camera, how far its clock was off, how many clips it locked by sound.
- Then it builds the timeline. Spot-check one moment with several angles, like a clap, a kiss or a toast.

You can keep talking to it in plain English: "keep the camera audio only where there's no mic," or "keep the drone clips lime color all day." 


# Finalkurt AI workflow

Director bot — for a new event (concert, church, wedding, birthday), kicks off and runs the right bots end to end: sync → cut → transcribe → story
 
## resolve-event-pipeline — Director bot
The one you call first. It picks a route and runs the worker bots in order in the same session (each needs the previous one's work files, so it doesn't farm them out to subagents).
- Triggers: "new project", "new concert/service/wedding/birthday", "get the bots on it", "sync and cut this", "build me the story"
- Routes: concert / church service / performance → Sync, then Cut (Transcribe optional for sermon or MC quotes). Wedding / birthday / party → Sync (only if there's separate clean audio or multicam), then Transcribe, then Story, with Cut optional on speeches or the first dance. Interview / documentary → Sync if dual-system, then Transcribe, then Story.
- Kickoff: one question block, max 4 — event type and names, where the media is, the audio layout, and the deliverable. Say "go" and it uses defaults and tells you what it picked.
- Preflight: confirms Resolve is running and the project name, inventories the bins and reads them back to you to correct assumptions (it carries the note that last time "one long mix file" turned out to be 9 per-artiste files), and sets the timeline frame rate before any timeline exists — UK 25/50p means 25. If timelines already exist at the wrong rate it tells you, because that can't be changed afterwards.
- Guardrails: never deletes or overwrites your existing timelines or tracks without asking; new timelines get version suffixes. If the PC isn't reachable or Resolve isn't running it says so and stops rather than guessing.
Long stages run as background workers on the PC — audio extraction is roughly 15–20 min for 6 hours of camera audio — with a short progress message after each stage. Work files stay in
 
 
## resolve-multicam-sync — Sync bot
Takes a project with camera bins plus a bin of clean mix/master WAVs and builds one timeline per mix file: cams on V1/V2/V3… video-only (no camera audio), the mix WAV on A1 named "Mix", every camera clip placed to within a frame and trimmed to that mix file's length.
- Proven on: the Adedoyin concert — 3 cams, 87 clips, 9 mix files, everything inside 20 ms.
- How: inventories the media pool into a manifest (roles from bin names — "mix"/"master" or audio type; "cam"+number sets the track), extracts audio with ffmpeg to 8 kHz mono, builds onset envelopes at 200 per second, cross-correlates every camera clip against every mix file, derives each camera's clock offset from the confident matches using Date Created as wall clock, then refines each clip in a ±3 s window against a combined session envelope.
- Flags on the timeline: orange clip plus a red "CHECK SYNC" marker where it isn't confident, yellow for borderline. It reports clips that fall outside any mix (between sets, pre-show) and mix files with no footage at all. A clip spanning two mix files goes into both, trimmed.
- Sanity checks before building: that the mix files share one session clock, that each camera's offset spread is within about ±1.3 s, and that a mix stamped at the default 01:00:00:00 isn't orphaned. A single long mix for a whole show needs set boundaries from you — times or markers.
Verifies no overlaps, placement within half a source frame, and track names, then reports a table plus the clips you must eyeball.
 
## resolve-multicam-roughcut — Cut bot
Takes the Sync bot's timelines and adds a new track above the cameras called "Cut" (V4 for 3 cams) holding a rough edit that alternates angles, with every cut snapped to the strongest onset in the mix. V1–V3 stay untouched underneath so you can swap any shot.
- Defaults: medium pace, 4–8 s shots, cut on V4 in the same timeline. It only asks if this one differs — slow 6–12 s, fast 2–5 s, or mixed by set (slow for worship and sermon, faster for praise) — or whether you want separate "– CUT" timelines.
- Planner: scores beats from the mix envelope, picks angles with a seeded random choice, avoids repeating the previous angle, respects each clip's availability with a margin off the ends, and lands each cut on the best beat near the target shot length.
Verifies that every cut shot frame-matches the clip underneath it, expecting zero mismatches and zero overlaps; gaps only where no camera was rolling.
- Honest caveats in its report: long holds where only one camera rolled, gaps with no picture, short shots at clip ends, and that the cut is timing-based only — framing and focus aren't judged. It offers a pace change or a re-roll on a new seed, and asks before clearing an existing V4 cut (or makes "Cut v2").
 
 
## resolve-footage-transcribe — Transcribe bot
Turns every clip into one machine-readable footage_transcript.xml — timecoded segments, speakers and individual words per clip — which the Story bot then reads. Works for weddings, birthdays, documentaries and interviews.
- Uses Resolve's native transcription with speaker detection, per clip or per whole bin, saving the raw result per clip and retrying empties once before marking them "no-speech".
- Prefers your clean audio: if the Sync bot has run, it transcribes the mix/lav files rather than camera scratch and maps the text back to the cameras through the sync plan.
- Fallback if native transcription fails: faster-whisper on the PC (medium, or large-v3 with a GPU) with word timestamps, as a background worker writing the same shape.
- Optional visual tags: ffmpeg makes one contact sheet per clip — 12 frames tiled 4×3 across its length — which get staged up and looked at, producing 3–8 tags and a one-line description from a fixed vocabulary (getting-ready, rings, vows, kiss, speech, first-dance, cake, drone, and so on). Default on for wedding and birthday story builds.
- Language: English default. Yoruba or Pidgin speeches get flagged for you to listen to, since accuracy drops.
The XML is saved in the work dir and beside the footage in a _story folder — it asks before writing into your media folders the first time — with a JSON twin for fast loading. Reports speech minutes, no-speech and low-confidence clips, and your top 5 speakers by talk time.
- One flag worth knowing: the skill marks its own first-run behaviour as untested in your setup — whether Resolve's transcription blocks past the 60 s scripting limit — and is written to record what worked so it can be updated.
 
 
## resolve-story-builder — Story bot
Reads the transcript XML, chooses the moments that tell the story, builds a first-assembly timeline, and exports it. It describes itself plainly as a strong first assembly, not a finished film.
- Delivers: story.xml (the edit plan), a timeline "STORY – <type> v1" (never overwriting — it bumps the version), and both story_fcpxml.fcpxml (FCPXML 1.10) and story_fcp7.xml.
- Brief: wedding or birthday with names; teaser 60 s, highlight 3–5 min (default) or full story 8–12 min; a music track from a "music" bin; tone emotional/cinematic (default), fun/upbeat, or faith-forward, which keeps prayers, blessings and worship moments.
- Two-stage selection: code builds the shortlist — segments of 5+ words, same-speaker fragments under 0.6 s apart merged into complete thoughts, scored on emotional keywords, a bonus for laughter or applause landing right after and for clean sentences, a penalty for fillers and for running over 20 s — keeping the top ~60. Then the actual story choice is made by reading that shortlist alongside the visual tags. The skill explicitly says not to leave that to code.
- Arcs: a 9-beat wedding highlight (hook → getting ready → ceremony build → vows and rings as the peak → kiss and exit → reception entrance → speeches → first dance → closing) and an 8-beat birthday (hook → decor → celebrant entrance → tributes → prayer if faith-forward → cake and candles → dancing → closing).
- Rules: complete thoughts trimmed on word boundaries with 4–8 frame handles, never cutting mid-word; vary speakers; soundbites make up 40–55% of the target with B-roll and music filling the rest; hold the emotional peak and don't cut in during vows. Non-English or low-confidence clips are used for their pictures only and listed for you to audition.
- Layout: V1 soundbite picture, V2 B-roll video-only, A1 soundbite audio, A2 music, with a blue marker per beat and yellow markers on low-confidence lines. Music goes down full length with a note that you should duck it — it won't fake a mix.
Verifies that the first word lands where the transcript says within half a second, spot-checking three against the word timings, no overlaps, and total length within ±15% of target.
 
 
## resolve-colour-grade — Grade bot
Builds a node-based grade for S-Log3 from a reference look and applies it across a timeline. Built on the FKF "Golden Hour" grade you made for Heal Me (A7S3, S-Log3 / S-Gamut3.Cine, 25p, UHD).
- Triggers: "colour grade this", "build a power grade", "match this reference", "grade the timeline"
- The tree: colour-space transform in → per-clip exposure → white-balance trims → a parallel block of film contrast, golden tone and skin/greens → mixer → saturation and curves → glow → transform out to Rec.709 → grain. The fixed transforms are DCTLs; nodes 03, 08, 09 and 11 are deliberately left empty for your WB trims, curves, Glow/Halation and Film Grain.
The awkward constraint it works around: Resolve's API can't add nodes, wire them or label them. So the tree gets built by actually driving the Color page — Alt+S and Alt+P in the node graph through computer use — and everything else goes through the API. If window focus keeps getting stolen it skips the cosmetic labels and tells you.
- It holds the real maths for rebuilding the DCTLs: the S-Log3 and DaVinci Intermediate curves, the gamut matrices, luma weights, the filmic tone map and the golden-tone offsets. So "warmer" means raising specific offsets, "punchier" means contrast 1.35, "moodier" means tone-map scale 0.45 — not guesswork.
- Iterates cheaply first: pulls 5–6 representative frames via ffmpeg (bright exterior, backlit, dark interior, skin close-up, saturated wardrobe), runs a numpy copy of the whole pipeline in the cloud, makes before/after mosaics, tunes the constants against your reference, then writes the DCTLs.
- Per-clip exposure pass: samples 3 frames per clip, solves for an offset putting median display luma at 0.40 while protecting highlights, then applies a damped, clamped version on node 02. It warns that bright-window interiors come out too dark and need a hand override.
Delivers a PowerGrade still in the "FKF Looks" album, a .drx export, the grade on every clip on every video track, and check frames. Never resets an existing multi-node grade without asking. For other cameras, only the first node changes — DJI D-Log M, Canon Log 3, V-Log.
 
 

 

