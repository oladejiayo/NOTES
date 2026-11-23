

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
