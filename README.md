# NoAppVoiceRoom
Setup a voice room without downloading any app. Build a server on your own computer.

# 語音房 · Voice Room

A self-hosted voice chat room that runs on your own Windows PC. Share a link, friends click it, and they're in the call. They don't need an app, an account or a download. As the host you can share your screen, and everyone can set how loud each person sounds to them.

---

## Features

- **One-click start**: double-click `start.bat` and the room is up.
- **Join with a link**: friends open the link in any modern browser, type a name and join.
- **Voice chat**: everyone can hear everyone. A green border shows who's talking.
- **Per-person volume**: each person has their own volume slider (0–200%) and a mute button. These only change what *you* hear.
- **Master volume**: one slider for everything you hear.
- **Deafen (拒聽)**: one button to stop hearing everyone, including screen audio.
- **Mic toggle**: mute or unmute yourself. If someone denies microphone access, they join as a listener.
- **Host screen sharing**: the host can share a screen, window or tab, with system audio. Viewers can hide it (不看) or go fullscreen.
- **Keeps running in the background**: switching tabs or minimizing the window doesn't drop the call.
- **Installable as an app (PWA)**: an "📲 安裝 App" button adds it to the home screen or desktop.
- **End button**: the host's **⏻ 結束整個 App** button disconnects everyone and shuts down the server and the tunnel.
- **Self-contained**: nothing is installed on your system. Everything lives inside this folder.

---

## Quick start

1. Unzip the folder anywhere.
2. Double-click **`start.bat`**.
   - The **first run** downloads a portable Node.js and `cloudflared` into `runtime\` (about 90 MB, only once).
   - A separate Edge window opens with the **host page**. Its URL contains `?host=…`, so **don't share that one**.
3. The host page shows a public link like `https://xxxx.trycloudflare.com/?k=…`. Click **複製** (Copy) and send it to your friends.
4. Friends open the link, enter a name, and click **加入通話** (Join).

To stop: click **⏻ 結束整個 App** on the host page, or close the black console window.

---

## Folder layout

```
voice-room\
├─ start.bat          ← double-click to start
├─ uninstall.bat      ← removes downloaded runtime + data
├─ server.js          ← the server (Node.js built-ins only, no npm packages)
├─ public\            ← the web page (HTML/CSS/JS, PWA manifest, service worker, icons)
├─ runtime\           ← created on first run
│  ├─ node\              portable Node.js
│  └─ cloudflared.exe    creates the public https link
└─ data\              ← created on first run
   ├─ room-key.txt       the room key in your share link
   ├─ browser\           separate Edge profile for the host window
   └─ cloudflared\       cloudflared's home folder
```

---

## Isolation: what it does *not* touch

- **Node.js** is a portable zip in `runtime\node`. It doesn't run an installer, change PATH or write to the registry. If you already have Node.js installed, Voice Room neither uses it nor changes it.
- **cloudflared** lives in `runtime\`, and its home directory points at `data\cloudflared`.
- **Host window** uses its own Edge profile in `data\browser`, so your normal browser's history and cookies are untouched.
- **Firewall**: the server only listens on `127.0.0.1`, so Windows Firewall doesn't prompt and no rules are added.

**To remove it completely**, delete the folder. To keep the code but clear the downloads, run `uninstall.bat`.

---

## Configuration

Edit the `CONFIG` block at the top of `server.js`:

| Option | Default | Description |
|---|---|---|
| `port` | `41739` | Local port. It's an uncommon one; change it if something else is using it. |
| `useQuickTunnel` | `true` | Creates a free Cloudflare https link on every start. |
| `publicUrl` | `''` | Your own fixed URL, e.g. `https://voice.example.com`. If set, the quick tunnel is skipped. |
| `allowLan` | `false` | `true` allows same-Wi-Fi access over http when there's no tunnel. Windows will ask about the firewall and add a rule. Over plain http, guests can only listen, because browsers block microphones on non-https pages. |
| `iceServers` | Google + Cloudflare STUN | Add a TURN server here if some people can't connect (see Troubleshooting). |

**New share link:** delete `data\room-key.txt` and restart. The old link stops working.

---

## How it works

```
 Friend's browser ──https──► Cloudflare tunnel ──► cloudflared.exe ──► server.js (127.0.0.1:41739)
        ▲                                                                   │
        └──────────── WebRTC audio/video, peer-to-peer (not through server) ─┘
```

- **Signaling**: `server.js` relays only connection setup messages, using **long-polling** over plain HTTP. Every proxy and tunnel handles this reliably, and messages aren't lost on a brief disconnect.
- **Media**: voice and screen video go **directly between browsers** over WebRTC, in a full mesh where each person connects to every other person. Your PC and Cloudflare don't carry the media.
- **Presence**: someone who closes the page is removed within about 1 second if the browser's leave message arrives. If it doesn't, they're removed within about 10 seconds. Two things catch that case:
  - other participants report when their connection to that person drops
  - the server notices the person has stopped checking in

---

## Installing as an app

Open the **https** link, then:

- **Chrome / Edge (PC or Android)**: click **📲 安裝 App**, or use the browser menu → *Install app*.
- **iPhone / iPad**: open the link in Safari → **Share** → **Add to Home Screen**.

> ⚠️ The free quick-tunnel URL **changes every time you restart**, so an installed app will point to an old address. For an installed app that keeps working, set `publicUrl` to a stable address. Two ways to get one are a Cloudflare named tunnel on your own domain, or Tailscale Funnel. The room key in `data\room-key.txt` stays the same across restarts.

---

## Background audio

| Platform | Switch tab / minimize | Switch to another app | Close the page |
|---|---|---|---|
| Windows / macOS | ✅ keeps going | ✅ keeps going | leaves the call |
| Android (Chrome) | ✅ | ✅ usually keeps going | leaves the call |
| iPhone (Safari) | ✅ | ⚠️ iOS often pauses mic/audio, so tap **點這裡恢復** when you come back | leaves the call |

The iOS behavior is a platform limitation. A web page can't work around it.

---

## Troubleshooting

**A friend shows "連線中…" forever, or is connected but silent.**
Their network, often mobile 4G/5G, is probably blocking direct peer-to-peer connections. Add a TURN server to `iceServers`:
```js
{ urls: 'turn:your-turn-server:3478', username: 'user', credential: 'pass' }
```

**The host page says "沒有公開連結".**
`runtime\cloudflared.exe` failed to download. Close the window and run `start.bat` again. You can also download `cloudflared-windows-amd64.exe` from Cloudflare's GitHub releases, rename it to `cloudflared.exe`, and put it in `runtime\`.

**"Port 41739 已被占用" (port in use).**
Another copy is probably still running, so close it. Otherwise, change `port` in `server.js`.

**Friends can't hear my computer's sound while I'm sharing my screen.**
When the share picker appears, tick **Share system audio** (Windows Chrome/Edge) or **Share tab audio**.

**Node.js download failed.**
Check your internet connection and run `start.bat` again. The script uses Windows' built-in `curl` and `tar`, so it needs Windows 10 version 1803 or newer.

---

## Limits

- **Group size**: about 8 people or fewer is recommended. In a full mesh each person uploads their audio once per participant, and the host uploads the screen once per viewer.
- **Screen sharing**: works from desktop browsers. Phones can watch but can't share.
- **Uptime**: the room exists only while your PC and `start.bat` are running.

---

## Security notes

- Only people with the `?k=…` link can join. Treat the link like a password.
- Only the host page, opened with a token that's new each run, can share the screen or shut down the server.
- The server binds to `127.0.0.1`, so nothing on your network can reach it directly.
- Audio and video are encrypted end-to-end between browsers, which WebRTC requires.

---

## Requirements

- **Host**: Windows 10 (1803+) or 11, with Microsoft Edge (preinstalled). Internet is needed for the first-run downloads.
- **Guests**: any recent Chrome, Edge, Firefox or Safari, on desktop or mobile.
