<pre>
 _____
|_   _|
  | | ___  __ _ _ __ _   _ū
  | |/ _ \/ _` | '__| | | |
  | |  __/ (_| | |  | |_| |
  \_/\___|\__,_|_|   \__, |
                      __/ |
                     |___/
</pre>

# 怠竜 Tearyū's VidSnatcher

**Detect and download any video the web throws at you. HLS, DASH, AES-128 encrypted, fMP4 — all of it. No website needed. No "premium plan". No 47 tabs of ads.**

[![Made by The Lazy Dragon](https://img.shields.io/badge/Made%20by-The%20Lazy%20Dragon%20%E6%80%A0%E7%AB%9C-ff153f?style=flat-square)](https://github.com/The-Lazy-Dragon)
[![Chrome Extension](https://img.shields.io/badge/Type-Chrome%20Extension-00fff7?style=flat-square)](#)
[![Manifest V3](https://img.shields.io/badge/Manifest-V3-8e44ad?style=flat-square)](#)
[![HLS](https://img.shields.io/badge/HLS-supported-c080ff?style=flat-square)](#)
[![DASH](https://img.shields.io/badge/DASH-supported-c080ff?style=flat-square)](#)
[![AES-128](https://img.shields.io/badge/AES--128-decrypted-00c87a?style=flat-square)](#)
[![Version](https://img.shields.io/badge/Version-1.1.0-ff153f?style=flat-square)](#)

*"why would i ever pay for a video downloader bro" — me, after writing this*

---

## ⚡ What is this?

A Chrome extension that **detects and downloads any video the page is playing**, including:

- 🎬 **Direct files** — `.mp4`, `.webm`, `.mkv`, `.mov`, `.flv`, all of em
- 📡 **HLS streams** (`.m3u8`) — including the master playlist auto-pick-best-quality kind
- 📡 **DASH streams** (`.mpd`) — with `SegmentTemplate`, `SegmentTimeline`, `SegmentList`, the works
- 🔓 **AES-128 encrypted** HLS segments — decrypts on the fly
- 📦 **fMP4 segments** — pulls the `#EXT-X-MAP` init segment so the output actually plays
- 🌐 **Obfuscated embeds** — XHR/fetch hooks catch URLs even when the player tries to hide them

It hooks `XMLHttpRequest`, `fetch`, the `<video>` `src` setter, and `setAttribute` — runs in the page's own JS realm at `document_start`, *before any page script runs*. So by the time the player wakes up, we're already listening.

**What it WON'T do:** download from Netflix, Disney+, HBO, Crunchyroll, or any of the other 25 paid streaming platforms on the blacklist. Big companies, big lawyers. Not worth it. List is in `background.js` if you wanna see who I'm scared of.

---

## 🚀 How to Install

> Takes like 30 seconds.

**1.** Hit the green **`Code`** button → **`Download ZIP`** → extract it

**2.** Open Chrome and head to:

```
Extensions → Manage Extensions
```

*(or just slam `chrome://extensions` into the address bar)*

**3.** Flip on **Developer Mode** — toggle in the top right corner

**4.** Hit **Load unpacked** → select the `vid-snatcher` folder you just extracted

**5.** Pin the extension to your toolbar so the icon shows up

**6.** Go to literally any site with a video — open the popup — it's already detecting

> If you update the files later, just hit the 🔄 refresh icon on the extension card and reload the tab.

---

## 🎮 How to Use

The popup has **three tabs** and a **scan button**. That's it.

| Tab | Shows |
| --- | --- |
| **All** | Everything detected so far |
| **Direct** | `.mp4`, `.webm`, `.mkv` — stuff you can save with a single click |
| **Streams** | HLS / DASH — needs stitching |

**For direct files:** click `⬇ Download` → standard Chrome save dialog → done.

**For streams (HLS/DASH):** click `⬇ Stitch & Download` → it parses the manifest, pulls every segment in parallel (×6), decrypts if AES-128, stitches them, and hands you a single file. Live progress bar in the popup. Survives popup close — there's an offscreen document doing the work.

**The badge:**
- **Red number** = videos detected on this tab
- **Purple %** = global, a stream download is running

---

## 🐉 Why It's Different

I'm not saying it's better than every video downloader on the store. I am saying every video downloader on the store is:

1. Paid
2. Hidden behind 14 popups asking you to rate them
3. Sending your URLs to a server you don't own
4. "Free tier limited to 480p" — bro

This one is **local**. Everything happens in your browser. The segments never leave your machine. There's no server. There's no telemetry. There's not even an "about" page asking for a tip.

---

## 🔍 How Detection Actually Works

Five hooks, all installed in the page's own JS realm before any page script runs:

| Hook | What it catches |
| --- | --- |
| **`XMLHttpRequest.open`** | Legacy XHR-based players |
| **`fetch()`** | Every modern player on earth |
| **`HTMLMediaElement.src` setter** | When a script assigns `video.src = "..."` |
| **`Element.setAttribute('src', ...)`** | Same as above but via setAttribute (some players prefer this) |
| **`MediaSource.addSourceBuffer`** | Tells you the page is using MSE so you know the real segments are above |

Plus a **DOM scanner** that runs on load and on every mutation — catches `<video>` elements that already exist, `<a href="...mp4">` direct links, and late-injected players.

Plus a **network-level `webRequest.onResponseStarted` listener** in the service worker — catches anything that slipped past the above based on the response's `Content-Type` header.

If a URL got loaded by the page in any way, we know about it.

---

## 📦 Streaming Engine

Lives in a **MV3 offscreen document** so it keeps running when you close the popup. Because Chrome's service workers fall asleep after 30 seconds, and trying to download a 2-hour HLS stream in 30 seconds is, uh, not a thing.

**HLS** (`.m3u8`):
- Master playlist → auto-picks the highest bandwidth variant
- Pulls `#EXT-X-MAP` init segment if present (required for fMP4 streams)
- AES-128 keys cached, IV computed correctly (explicit or from segment index)
- Detects whether segments are MPEG-TS (`.ts`) or fMP4 (`.m4s`/`.mp4`) and picks the right output extension/MIME
- Refuses to engage with SAMPLE-AES (that's DRM territory, browser can't decrypt that)

**DASH** (`.mpd`):
- Resolves `BaseURL` chain (MPD → Period → AdaptationSet → Representation)
- Picks highest-bandwidth video Representation
- Supports `SegmentTemplate` with `$Number$`, `$Time$`, `$RepresentationID$`, `$Bandwidth$`, and `%0Nd` zero-padding
- Supports `SegmentTimeline` for VOD with non-uniform segment durations
- Supports `SegmentList` and `SegmentBase`
- Computes total segment count from `mediaPresentationDuration` when needed

**Both:**
- **6 segments downloaded in parallel** instead of one at a time — way faster
- Per-segment retry (3 attempts with exponential backoff)
- Tolerates up to 5% segment failures before giving up
- All segments held as `ArrayBuffer`s, stitched into one `Blob`, handed to `chrome.downloads.download`

---

## 🚫 The Blacklist

These domains are hard-blocked. No detection, no hooks, no popup activity. Look like the site isn't even there:

```
netflix.com, primevideo.com, amazon.com, disneyplus.com, hulu.com,
max.com, hbomax.com, peacocktv.com, paramountplus.com, appletv.apple.com,
crunchyroll.com, funimation.com, mubi.com, curiositystream.com,
discoveryplus.com, espnplus.com, starz.com, showtime.com, mgmplus.com,
hotstar.com, jiocinema.com, sonyliv.com, zee5.com, tv.apple.com,
nowtv.com, britbox.com, acorn.tv, shudder.com, criterion.com, tv2.no
```

If your favorite streaming service isn't on the list, that's because the list is for **commercial DRM-protected platforms specifically**. The blacklist is at the top of `background.js`, `content.js`, `inject.js`, and `popup.js` — search for `BLACKLIST` and modify if you want, that's your business.

---

## 📂 What's in the Folder

```
vid-snatcher/
├── manifest.json          ← MV3 config
├── background.js          ← service worker · webRequest, state, badge, downloads
├── content.js             ← isolated-world bridge · DOM scan
├── inject.js              ← MAIN-world hooks · XHR, fetch, video.src
├── offscreen.html         ← offscreen document loader
├── offscreen.js           ← HLS/DASH download engine — AES-128, parallel fetch
├── popup.html             ← UI
├── popup.js               ← UI logic
└── icons/                 ← 16, 48, 128
```

---

## 📋 Changelog

### v1.1.0 — The "It Actually Works Now" Update

- 🔴 **CRITICAL FIX** — offscreen document had an inline `<script>` tag. MV3's default CSP (`script-src 'self'`) blocks inline scripts in extension pages, so the *entire HLS/DASH download engine never ran*. Click "Stitch & Download" → nothing happens. Moved to external `offscreen.js`. **This was the killer bug.**
- 🔴 **Removed `window.eval` and `document.write` overrides** — they broke any site with strict CSP and any framework that legitimately uses eval (Webpack, some SPAs). The XHR/fetch hooks catch the URLs anyway.
- 🟠 **HLS fMP4 init segment support** — parses `#EXT-X-MAP` and prepends the init segment. Without this, fMP4 HLS streams downloaded as broken files.
- 🟠 **DASH parser rewrite** — proper XML parsing, BaseURL chain resolution, `SegmentTimeline` support, `$Number%05d$` padding, AdaptationSet filtering for video tracks.
- 🟠 **Parallel segment downloads** — 6 concurrent fetches instead of serial. ~5× faster on long streams.
- 🟠 **Per-segment retries** with exponential backoff. CDN hiccup no longer kills the whole job.
- 🟠 **Correct output extension** — fMP4 streams now save as `.mp4` instead of `.ts`. Players actually open them.
- 🟠 **Service worker state persistence** — `detectedVideos` now lives in `chrome.storage.session`. SW falls asleep, wakes up, your detected list is still there.
- 🟠 **`mediasource://` fake URLs filtered out** — were appearing in the list as bogus "MP4" downloads. Now reported as info-only.
- 🟡 Badge logic fixed — global download % no longer permanently overwrites per-tab counts.
- 🟡 Offscreen creation race condition fixed (multiple stream downloads at once no longer race).
- 🟡 Removed unused `scripting` permission. Removed weird empty `{icons}` folder.
- 🟡 Double-install guard on inject.js (was getting installed twice in some iframe scenarios).
- 🟡 Setattr hook narrowed to `<video>`/`<audio>`/`<source>` only — was running on every setAttribute on the entire page.

### v1.0.0 — Alpha

- Initial release. Looked great. Didn't work.

---

## ⚠ Known Stuff

- **No audio mux for video-only HLS/DASH** — if a stream has video and audio as separate renditions (common on YouTube-style adaptive streams), you'll get video-only. Browser-side muxing without a remuxer like ffmpeg.wasm is a whole project on its own. On the roadmap, maybe.
- **No `SAMPLE-AES` / Widevine / FairPlay / PlayReady support** — that's DRM, the browser can't decrypt those keys even if it wanted to. Anything legitimately DRM-protected is out of scope and always will be.
- **Some sites blob: the entire video** — if a page is using MediaSource with a single concatenated blob, there's no segment URL to grab. The popup will flag this with a "MediaSource" badge. The actual `.m3u8` or `.mpd` should still appear above it — that's what to download.
- **Cross-origin segments without proper CORS** may fail to fetch from the offscreen context. Most CDNs allow this; some don't.
- **Tested on Chrome / Edge / Brave.** Firefox doesn't support MV3 offscreen documents the same way, so the stitcher won't work there yet.

---

## 🐉 About the Dev

**The Lazy Dragon** · `怠竜 Tearyū`

BSc IT student who built a video downloader instead of studying for finals.

- GitHub: [@The-Lazy-Dragon](https://github.com/The-Lazy-Dragon)
- Discord: `this.isnt.craig_`

### Also check out:

**[Tearyu Deadshot Crosshair](https://github.com/The-Lazy-Dragon/Tearyu-Deadshot-Crosshair)** — A crosshair extension for deadshot.io that does way more than a crosshair extension should. 15+ shapes, custom images, orbit modes, hit markers, the whole circus.

**[NavalStrike](https://github.com/The-Lazy-Dragon/NavalStrike)** — Battleship. But neon. But with 25 ship classes, 35 achievements, a Konami code, and hard AI that actually hurts. One `.html` file.

**[Lazy Dragon's TicTacToe](https://github.com/The-Lazy-Dragon/lazy-dragons-tictactoe)** — TicTacToe. But neon. But with achievements and an unbeatable minimax AI. Also one `.html` file.

---

**⭐ Star the repo if you finally got that video off the random site that wouldn't let you**

*Built with zero frameworks, no dependencies, and the firm belief that downloading something playing on your own screen should not require a subscription*
