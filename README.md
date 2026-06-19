# Dubz Final

Clean video-only dubbing pipeline. No YouTube, no yt-dlp, no WARP, no lip-sync.

## How it works
1. Drop **video files** into the Drive inbox folder.
2. GitHub -> **Actions** tab -> **Dubz Final** -> **Run workflow**.
3. Every video is dubbed in parallel (max matrix fan-out) via the dubbing engine.
4. Dubbed files land in **Dubz/DubDone**. Originals move to `<inbox>/archive`.

## Folders
- Inbox: set per account (Dubz/DubInbox or Dubz/DubInbox2)
- Output: Dubz/DubDone (universal)

## Settings (Run workflow form)
src, target, genre, speakers, chunk, workers, tries - all have sane defaults
(Hindi -> English, monologue, 1 speaker).

## Secret required
- `RCLONE_CONF` - your rclone config (Google Drive remote named `gdrive`).

No cron. Manual run only.
