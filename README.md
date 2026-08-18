# VOID//PURGE — ADB module for Shevery

Advanced cleaner: app cache, orphaned files from uninstalled apps, empty folders, log files, duplicate files, and a one-tap background app killer.
Architecture: everything is **scanned** first, previewed, and only deleted after explicit confirmation — no destructive operation ever runs blind (except cache trimming, which is inherently safe since it uses Android's own official mechanism).

## Local testing before publishing

```bash
cd void-purge-en
zip -r ../void-purge-en.zip .
```
Then in Shevery → ADB Modules → Install ZIP → pick this file.

**Requirement:** module access mode must be **Full access**, or in **Custom** mode the **WebUI shell bridge** option enabled. Without this, a red warning appears at the top of the WebUI and no button will work (intentional — surfaces the problem clearly instead of failing silently).

## Pre-publish checklist

- [✅] Bridge status badge at the top shows ✓ green
- [✅] Tap "Clear cache", check the debug console output at the bottom
- [✅] Tap "Scan orphaned files" — verify the list really belongs to uninstalled apps, not installed ones
- [✅] Create a test duplicate file (`cp file1.txt Download/file2.txt`) and confirm the duplicate scanner detects it correctly
- [✅] Test "Kill background apps" with an empty exclude list, confirm Shevery itself (`com.hamondev.shevery`) is not killed
- [✅] Test every scan-scope field with an empty value or `/` — it should be rejected (safety guard)
- [✅] (Root devices) Run Corpse Finder, manually verify results are genuinely uninstalled apps
- [✅] (Root devices) Clear one app's cache, reopen that app, confirm it doesn't crash (only cache was cleared, not config)

## Why it's built this way

- **Cache cleanup uses `pm trim-caches`**, not manual `rm -rf` — the only official, permitted method that works both under plain ADB shell and root, without risking a running app.
- **Orphaned files** are detected by comparing `Android/data`/`Android/obb` folder names against `pm list packages` output — shell has read access to these paths even without root.
- **Deletions are batched into a single command** (not looped per-file) — `window.Shizuku.exec` is synchronous and blocks the UI while running, so batching is faster and reduces the chance of the UI hanging.
- **Kill Background Apps runs as fire-and-forget** (`(...) &` backgrounded on-device) — an earlier version waited for every single `am force-stop` to finish sequentially, which could take minutes with 100+ apps installed and froze the WebView long enough to trigger an ANR. Now the call returns almost instantly and the kills happen in the background.
- **Duplicate scanning is hard-capped at 1200 files per run** with a single combined hash+size pass — an earlier version had no cap and a ~10-minute timeout, which is exactly what caused multi-minute freezes on large folders.
- **Every path is quoted** with `shQuote()` — without this, a filename with spaces, quotes, or non-Latin characters could break the shell command or, worse, execute something unintended.
- **A hardcoded denylist of critical paths** (`/sdcard`, `/data`, `/`, ...) is checked in JS, independent of any bug in parsing `find`/`du` output.

## Known non-goals (deliberately not implemented)

Wiping a whole app's data, touching leftover APKs in `/data/app`, systemless overlays, or anything that could affect boot or other running apps. These are exactly the kind of features that occasionally break things even in mature tools like SD Maid — not worth the risk for a published module.
