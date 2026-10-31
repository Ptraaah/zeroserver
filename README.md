# ZeroServer

Kirim file langsung dari browser ke browser. Tanpa server file, tanpa akun, tanpa batas ukuran dari layanan.

Send files directly from one browser to another. No file server, no account, no size cap set by any service. (English summary below.)

## What it is

ZeroServer does the job people know from WeTransfer — pick files, share one link, the other side downloads — with the opposite architecture: there is no server holding your files. The sender opens one web page, picks files, and gets a link and a QR code. The receiver opens the link and the files stream straight from the sender's device to the receiver's device over an encrypted WebRTC data channel.

No account on either side. No upload waiting time — sending starts the moment the receiver accepts. No expiry clock — a link lives exactly as long as the sender keeps the tab open, and the page says so plainly. When the transfer finishes, both sides see a SHA-256 check confirming the received file is byte-for-byte identical to the original.

The interface is Indonesian by default with an English toggle, and it is mobile-first: it is built for low-end Android phones on real-world mobile networks.

## No-server architecture

- The whole product is a single static HTML file. It loads two open-source libraries from a CDN (PeerJS for signalling, qrcode-generator for the QR code) and otherwise runs entirely in the browser.
- PeerJS's public broker introduces the two browsers to each other (signalling only). After that, file data never touches any server: it flows peer-to-peer over a DTLS-encrypted WebRTC data channel, using public STUN servers to find a path.
- Files are sent in 256 KB chunks with flow control. The receiver hashes every chunk as it arrives, so the final integrity check covers the exact bytes received.
- On Chrome and Android, received files stream straight to disk through the File System Access API. Other browsers hold the file in memory and offer it as a download when complete.
- The page includes a small connection-test panel: it shows whether the connection ran direct or via relay, how much moved, at what speed, and copies a plain-text result you can share for debugging.

## Self-host in one step

Copy `index.html` to any static host — GitHub Pages, Cloudflare Pages, Netlify, a plain web server, even a USB stick served locally. There is nothing to install, no build step, no backend, no database. That is the whole deployment.

## How a transfer works

1. The sender picks one or more files and presses "Buat link kirim" (Create send link).
2. The page creates a one-time peer ID and shows a link plus QR code. The sender shares the link, e.g. via WhatsApp.
3. The receiver opens the link, sees the file list and total size, and presses "Terima file" (Accept files). If the browser supports it, they pick where each file should be saved.
4. Files stream device-to-device with progress, speed, and time remaining shown on both sides.
5. Each file is verified with SHA-256 at the end. A green badge means the copy is identical to the original.

## Known limits

- **The sender must stay online.** There is no store-and-forward: if the sender closes the tab or loses connection, the transfer stops. ZeroServer is a live handoff tool — best while you are on a chat or call with the other person.
- **Some networks will not connect in v1.** Version 1 uses STUN only, with no relay (TURN) server. Most home and mobile connections connect directly, but strict corporate firewalls and some carrier NAT setups can block a direct path. If a connection fails, try another network (e.g. switch between mobile data and Wi-Fi).
- **Safari and Firefox have large-file limits.** Browsers without the File System Access API hold incoming files in memory, so a file over roughly 300 MB can crash the tab. For big files, receive in Chrome on Android or desktop Chrome.
- One receiver per link in v1, and no transfer history or accounts — by design.

## Run a connection test

1. Open the page on device A, pick a test file, create a link.
2. Open the link on device B (a different network is the interesting case: mobile data vs home Wi-Fi, different carriers, etc.) and accept.
3. When the transfer finishes, press "Salin hasil tes" (Copy test result) on either device.
4. Paste the copied summary wherever test results are collected. It contains the time, the device role, whether the path was direct or relayed, browser, total size, duration, average speed, and the result — everything needed to judge how well connections work on real networks.

## Licence

MIT — see [LICENSE](LICENSE). Copyright (c) ptraaa.

---

### English summary

ZeroServer is a single-file, open-source web page for sending files directly between two browsers over WebRTC. No file server, no accounts, no service-imposed size cap, and an end-to-end SHA-256 integrity check on every file. The sender must stay online during the transfer; v1 is STUN-only, so a minority of networks cannot connect directly; Chrome/Android is the best-supported receiving browser for very large files. Self-hosting is copying one HTML file to any static host.
