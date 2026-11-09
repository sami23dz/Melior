# MelioReader

A fast, private manga reader for Android. It reads from pluggable sources, keeps your library and progress on the device, and works offline for anything you download.

**Version:** 0.1.17 · **Min Android:** 6.0 (API 23) · **Languages:** English, Arabic (RTL)

## Features

- **Search across sources.** One query searches every enabled source and groups the same title together. It forgives typos ("hormiya" finds Horimiya).
- **Library and progress.** Continue where you left off, unread counts, and undo when you remove a series.
- **Reader.** Webtoon (continuous) and paged modes, zoom, and right-to-left support.
- **Downloads.** Save chapters for offline reading, with a foreground service for long queues.
- **Sources as plugins.** Each source is a small JavaScript file that runs in a sandboxed iframe. Bundled sources: 3asq, Mangalek, Mangalink, MangaLionz, MangaSpark, MangaStarz, MangaSwat, Team X.
- **Interface language.** Switch between Auto, English and العربية from Home or Settings.
- **Data saver.** Optional image resizing for slow connections.
- **Announcements and in-app updates.** Driven by a JSON file you host (see `announcements.example.json`).
- **MelioShield.** Library, progress, settings and source code are encrypted on disk with a key from the Android keystore. The app also warns if it was signed with a different key than the official one.

## Project layout

```
android/    Java shell: WebView host, native HTTP, storage, downloads, updater
www/        The app UI (plain JS/CSS, no framework)
  js/         screens and core logic
  engines/    shared source engines (e.g. Madara)
  plugins/    one file per source; index.json is generated
tools/      build.sh, bundle.js, update-entry.js, Announce Studio
tests/      unit, e2e and UI tests, plus plain-Java native tests
dev/        mock of the Android bridge for browser testing
```

## Build

Needs: `node`, `javac`, `aapt2`, `zipalign`, `apksigner`, a dex compiler (`d8` or `dalvik-exchange`), and an `android-34` `android.jar`.

```bash
# one-time: get the platform jar
mkdir -p /usr/lib/android-sdk/platforms/android-34
curl -L -o /usr/lib/android-sdk/platforms/android-34/android.jar \
  https://raw.githubusercontent.com/Sable/android-platforms/master/android-34/android.jar

bash tools/build.sh                 # signed APK at build/melioreader.apk (ProGuard on)
PROGUARD=0 bash tools/build.sh      # skip code shrinking/renaming
```

Set `ANDROID_JAR` if the jar lives elsewhere. `build.sh` regenerates `www/js/bundled.js` and `www/plugins/index.json` from the files in `www/plugins/`.

**Signing:** the script signs with `tools/melioreader-release.jks`. Use your own key and keep it and its password out of the repository.

## Tests

```bash
node tests/unit.js        # plain Node
bash tests/native.sh      # plain-Java tests (needs javac)
node tests/e2e.js         # browser tests (needs Playwright + Chromium)
node tests/ui.js
```

The browser tests run the web app against `dev/mock-native.js`, so they don't exercise the real keystore, downloads or WebView.

## Adding a source

Drop a file in `www/plugins/` that calls `registerSource({...})`, then run `node tools/bundle.js`. For WordPress "Madara" sites, an existing file is a good template:

```js
registerSource(Engines.madara({
  id: 'example', name: 'Example', lang: 'ar', version: 1,
  baseUrl: 'https://example.com',
  mangaSub: 'manga', chapterMode: 'manga-ajax'
}));
```

A source can implement `popular`, `latest`, `search`, `details` and `pages`.

## Releasing an update

1. Bump `versionName` and `versionCode` in `android/AndroidManifest.xml`.
2. Build, then upload the APK somewhere reachable over https.
3. Run `node tools/update-entry.js build/melioreader.apk <https-url> "notes" "ملاحظات"` and paste the printed block into your announcements file.

## Notes

- Sources are third-party sites. MelioReader doesn't host or control any content.
- Network access is https-only for public hosts; plain http is accepted only for addresses on your local network.
- No account, no analytics. The optional data saver sends image URLs to a third-party resizer, which is disclosed in Settings.
- 
