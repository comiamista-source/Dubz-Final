# Dubz-Final - Complete Setup & How It Works

Cloud video dubbing pipeline. Drop a video into a Google Drive folder, press one
button on GitHub, and a dubbed (e.g. Hindi -> English) version appears in your
output folder. All processing runs on GitHub's free servers - nothing runs on
your PC.

Repo: github.com/comiamista-source/Dubz-Final

---

## 1. THE BIG PICTURE

```
   You drop a video           GitHub Actions does the work            Result
  ┌──────────────────┐       ┌──────────────────────────────┐     ┌──────────┐
  │ Drive/Dubz/      │       │  scan  -> finds videos        │     │ Drive/   │
  │   DubInbox2/     │ ────> │  dub   -> 1 job per video     │ ──> │  Dubz/   │
  │   yourvideo.mkv  │       │         (chunk, dub, rebuild) │     │  DubDone/│
  └──────────────────┘       └──────────────────────────────┘     └──────────┘
        (input)                    (cloud, free runners)            (output)
```

Three Drive folders (all under a `Dubz/` folder on your Google Drive):
- `Dubz/DubInbox2`         - you drop videos here
- `Dubz/DubInbox2/archive` - originals are moved here after processing
- `Dubz/DubDone`           - dubbed videos land here

---

## 2. ONE-TIME SETUP (already done, for reference / rebuild)

If you ever rebuild on a fresh repo, this is what must exist:

### a) The repo files
- `dub_video.py`          - orchestrator: split, dub, rebuild, mux
- `engine/dub_engine.py`  - client for the dubbing API (Sarvam)
- `engine/requirements.txt`
- `.github/workflows/auto_dub.yml` - the GitHub Actions workflow

### b) Repo must be PUBLIC
Public repos get unlimited free Actions minutes. A long batch costs nothing.
Private repos have a monthly minute quota and then charge.

### c) Secrets (Settings -> Secrets and variables -> Actions)
- `RCLONE_CONF`  (required) - your rclone config so the runner can read/write
                              your Google Drive. This is the whole contents of
                              your rclone.conf with the [gdrive] remote.
- `DUB_BASE_URL` (optional) - overrides the dubbing endpoint if it ever changes.

NOTE: Dubz-Final dubs uploaded FILES. It does NOT download from YouTube, so it
does NOT need YouTube cookies. (That's the separate local-dub-auto pipeline.)

### d) rclone remote name
The workflow calls `gdrive:` - your rclone remote must be named `gdrive`.

---

## 3. DAILY USE

1. Put one or more video files into `Dubz/DubInbox2` on Google Drive.
   Supported: .mp4 .mkv .mov .webm .avi .m4v .ts
2. Go to: github.com/comiamista-source/Dubz-Final -> Actions tab
3. Click "Auto Dub from Drive" -> "Run workflow".
4. (Optional) fill the form fields - or leave them at defaults.
5. Press the green Run button.
6. Wait. Dubbed files appear in `Dubz/DubDone`. Originals move to the archive.

There is NO schedule/cron - nothing runs unless you press Run.

### Run-form options (all optional, with defaults)
| Field    | Default       | Meaning                                            |
|----------|---------------|----------------------------------------------------|
| src      | Hindi         | source language of the video                       |
| target   | English       | language to dub into                               |
| genre    | monologue     | content type (affects voice style)                 |
| speakers | 1             | number of speakers in the video                    |
| chunk    | 60            | seconds per chunk                                  |
| workers  | 8             | how many chunks of one video dub at once           |
| tries    | 5             | retries per chunk before it gives up               |
| skip     | (empty)       | ranges to KEEP ORIGINAL audio (see section 5)      |
| inbox    | Dubz/DubInbox2| input folder                                       |
| outbox   | Dubz/DubDone  | output folder                                      |

---

## 4. HOW IT WORKS (step by step)

### Stage 1 - scan job
- Lists files in `DubInbox2`, keeps only video extensions.
- Builds a "matrix": one parallel job per video.
- CAP: only the first 10 videos are taken (see Limitations).

### Stage 2 - dub job (runs once PER video, in parallel)
For each video the runner:
1. Downloads the video from Drive to the runner.
2. SPLIT: ffmpeg cuts the video into `chunk`-second pieces (default 60s),
   stream-copied (fast, no re-encode). Falls back to re-encode if needed.
3. DUB: every chunk is sent to the dubbing engine IN PARALLEL (up to `workers`
   at a time). Each chunk is uploaded, dubbed, and the dubbed piece downloaded.
   Failed chunks retry up to `tries` times.
4. REBUILD AUDIO: all dubbed chunks are stitched into one full-length audio
   track, each anchored at its EXACT original timestamp (so nothing drifts and
   the ending stays in sync). Missing chunks become silence (not a hard fail).
5. MUX: that new audio is laid onto the ORIGINAL video stream (video copied
   untouched, only audio replaced).
6. UPLOAD: the finished `<name> - English DUB.mp4` is copied to `DubDone`,
   and the original is moved to `DubInbox2/archive`.

### Parallelism - two levels
- max-parallel (workflow) = how many VIDEOS dub at once (now 10).
- workers (per video)     = how many CHUNKS of that video dub at once (8).
  Total load on the dub engine = videos x workers. 10 x 8 = up to 80 chunks
  in flight. That is heavy - see Limitations (engine rate limits).

### Resume / partial recovery
The split chunks live in a `work_<name>/chunks` folder on the runner. Finished
chunks are cached as `dub_XXX.mp4`. If you re-run `dub_video.py` pointed at a
chunk folder, it keeps finished chunks and only redoes the missing ones. (On the
cloud the runner is wiped between runs, so this mainly helps when running
locally.)

---

## 5. SKIP RANGES - don't dub the intro/outro song (NEW)

The dubber treats music like speech and mangles songs. Use `skip` to keep the
original audio for those parts.

Format: comma-separated ranges. `end` means end of video.
- `0-30,end-45`  keep first 30s AND last 45s as original
- `end-60`       keep the last 60s (outro song)
- `90-120`       keep 90s..120s as original
Everything OUTSIDE the ranges is dubbed normally.

CLI: `python dub_video.py video.mp4 --skip 0-30,end-45`

How it works: a chunk that overlaps a skip range by >=50% keeps its ORIGINAL
audio instead of being sent to the dubber.

---

## 6. RETRY LOGGING (NEW)

The run log now names the exact chunk on each retry:
```
chunk 47 retry 2/5: <reason>
chunk 47 FAILED after 5 tries: <reason>
```
Previously the cloud log only showed `93/96 done` with no per-chunk detail.

---

## 7. LIMITATIONS & KNOWN BUGS  (read this)

### Limitations
- MAX 10 VIDEOS PER RUN. The matrix is capped at 10 (`.[0:10]`, max-parallel 10).
  Drop more than 10 and only the first 10 process; the rest wait for your next
  manual run. (You asked for this cap.)
- DUB ENGINE IS THE BOTTLENECK, NOT GITHUB. All videos share one dubbing
  account. 10 videos x 8 workers = up to 80 chunks at once. If the engine rate-
  limits at that load, chunks fail and retry, which can make videos finish
  SLOWER (and burn more retries). If you see lots of "retry" lines, lower
  `workers` (e.g. 3-4) or run fewer videos.
- NO CRON / FULLY MANUAL. Nothing runs unless you press Run. (By design.)
- SKIP IS CHUNK-GRAINED. Skip ranges snap to chunk boundaries. With 60s chunks,
  `0-30` keeps the whole first 60s as original, not exactly 30s. For tighter
  intros, lower `--chunk` (e.g. 15-20). Trade-off: more chunks = more engine
  load.
- 350-MINUTE PER-VIDEO TIMEOUT. A single very long video that can't finish in
  ~5.8 hours will be cut off.
- ENGINE/ACCOUNT DEPENDENT. Dubbing quality, available languages, and speed all
  depend on the dubbing service behind DUB_BASE_URL. If that account runs out of
  credit or the service changes its API, dubbing fails until fixed.

### Known bugs / rough edges
- MUSIC IS DUBBED BY DEFAULT. Unless you set `skip`, intro/outro songs get sent
  to the dubber and come out mangled. The skip feature is the workaround, not a
  true music detector - there is NO automatic music/speech detection.
- MISSING CHUNKS BECOME SILENCE. If a chunk fails all its retries, that span of
  the final video is SILENT rather than failing the whole job. You may not
  notice unless you watch/listen. Check the log for "FAILED" lines.
- ELAPSED TIMER IS PER-JOB. Each video's job has its own elapsed clock. When the
  matrix moves to the next video you'll see the timer "reset" - that's a new
  job starting, not a restart of your video.
- PARTIAL OUTPUT IS NOT EXPORTED. A video only appears in DubDone when its whole
  job finishes (rebuild + upload happen at the very end). There is no half-dubbed
  file to grab mid-run; the work lives only on the runner until done.
- NON-ASCII / LONG FILENAMES. Output names are sanitised (odd characters become
  "_"). Usually fine, but very unusual filenames can look different from the
  source.
- SHARED DUB ACCOUNT ACROSS REPOS. If you run Dubz-Final and the other pipeline
  at the same time, they hit the same dubbing account and compete for capacity.

---

## 8. TROUBLESHOOTING

| Symptom                              | Likely cause / fix                          |
|--------------------------------------|---------------------------------------------|
| Run shows 0 videos                   | Wrong inbox folder, or file ext not in list |
| Only 10 of N videos ran              | The 10-video cap - run again for the rest   |
| Song/intro sounds mangled            | Use `skip` (e.g. 0-30,end-45)               |
| Many "retry" lines, slow finish      | Engine rate-limited - lower workers/videos  |
| A span of the video is silent        | A chunk FAILED all tries - see log, re-run  |
| Nothing in DubDone but job "done"    | Check the dub job log for the upload step    |
| Dubbing fails for every chunk        | Engine account/credit or DUB_BASE_URL issue |

---

## 9. WHAT IS *NOT* IN THIS REPO
- No YouTube downloading (no yt-dlp, no cookies). That is the separate
  local-dub-auto pipeline. Dubz-Final only dubs files you upload to Drive.