# MelioReader

A fast, private manga reader for Android. It reads from pluggable sources, keeps your library and progress on the device, and works offline for anything you download.

**Version:** 0.1.20 · **Min Android:** 6.0 (API 23) · **Languages:** English, Arabic (RTL)

## Features

- **Search across sources.** One query searches every enabled source and groups the same title together. It forgives typos ("hormiya" finds Horimiya).
- **Library and progress.** Continue where you left off, unread counts, and undo when you remove a series.
- **Reader.** Webtoon (continuous) and paged modes, zoom, and right-to-left support.
- **Downloads.** Save chapters for offline reading, with a foreground service for long queues. Pick chapters by hand (Details > download button > Select chapters; press and hold to pick a range), pause, resume or cancel one chapter or everything, retry failures, and see the queue and what is saved under Settings > Downloads > Manage. A paused chapter continues from the pages it already has.
- **Sources as plugins.** Each source is a small JavaScript file that runs in a sandboxed iframe. Bundled sources: 3asq, Mangalek, Mangalink, MangaLionz, MangaSpark, MangaStarz, MangaSwat, Team X.
- **Interface language.** Switch between Auto, English and العربية from Home or Settings.
- **Backup to a file.** Export saves a `.json` through Android's file screen (or copies the text); Import can load that file back. Nothing leaves the phone unless you move the file.
- **Backup addresses.** If a site stops answering (and the phone is online), the app tries the source's mirrors plus any addresses you add under Sources > the source, switches to the first that answers, and repeats the request. It tells you, and Reset goes back. A failed search is not repeated for a minute. Chapter images already listed under the old address keep that address.
- **Data saver.** Optional image resizing for slow connections. The default, "On mobile data only", shrinks pages while Android reports a metered connection and keeps full quality on Wi-Fi.
- **Load ahead.** The reader loads the pages on screen plus the next 3 (Settings > Reader > Load ahead, 1 to 12), in webtoon and paged modes, so stopping early does not download the rest of the chapter.
- **Announcements and in-app updates.** Driven by a JSON file you host (see `announcements.example.json`). When the file is on `raw.githubusercontent.com` the app reads it through GitHub's API (with ETag, so an unchanged file costs a tiny 304) because the raw address is cached for 5 minutes and a `?query` does not get around that; if the API is rate-limited or unreachable it falls back to the raw address. New posts show up within about a minute or two. No token is used or needed.
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

**Signing:** the script signs with `tools/melioreader-release.jks` (password `MelioReader`), which is kept in the project on purpose for personal builds. To sign with a key stored elsewhere, set `MELIO_KS` (path) and `MELIO_KS_PASS`. If this project ever goes on a public repo, take the key out first (see `.gitignore`): anyone holding it can sign an APK that Android and the app's own check accept as an official update.

## Release checklist

1. Raise both `android:versionCode` and `android:versionName` in `android/AndroidManifest.xml`.
2. `bash tools/build.sh` (needs the tests green: `node tests/unit.js`, `bash tests/native.sh`, `node tests/e2e.js`, `node tests/ui.js`).
3. Upload `build/melioreader.apk` somewhere with an https address.
4. `node tools/update-entry.js build/melioreader.apk <https address> "notes" "ملاحظات"` and paste the printed `update` block into the announcements file (or keep it when publishing with Announce Studio).

## Security model (short)

- **Sources** run in sandboxed frames that have no network of their own (a Content Security Policy blocks fetch, images, beacons and forms). The only way out is through the app, which applies the local-network block and the source's allowed sites. The app also refuses any web request the page itself makes except pictures through `img.melioreader`.
- **Pictures** are only served when the bytes really are an image; a check page or error page is never cached as one.
- **Stored data** (library, progress, settings, source code) is encrypted with a key held by the Android keystore. If a file can no longer be decrypted it is set aside (`*.unreadable`), never overwritten, and you are told once. **Not encrypted:** downloaded chapter pages and their manifests (which hold titles) in `files/dl`.
- **Updates** are checked against the hosted sha256, the version code, and (before Android is asked) the signing certificate. The hosted announcements file is not itself signed, so whoever controls that repository controls what the banner says.
- The tamper check is a warning, not protection: anyone modifying the app can remove it. Android's own signature check is the real barrier.

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
