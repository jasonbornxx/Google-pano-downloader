# 🌍 Earth360 Downloader

A single-file, client-side web app for downloading full-resolution 360° panoramas from Google Maps / Street View / Google Earth — no backend, no build step, just open `index.html`.

It detects the panorama type from a pasted URL, fetches the tiles (or the direct image), stitches them together, and saves a JPEG with GPS + Photo Sphere metadata embedded.

## Features

- **Smart URL detection** — paste a Google Maps, Street View, or `earth.app.goo.gl` link and the app extracts the pano ID automatically (via `!1s`, `panoid=`, `/p/` paths, or decoded Earth protobuf `data=` blobs).
- **Two panorama types supported:**
  - **Official Street View** (`sv`) — tiles are fetched at the maximum available resolution (up to 8K) from `streetviewpixels-pa.googleapis.com` and stitched client-side.
  - **User-uploaded 360° photos** (`CI...` / `AF1Q...` IDs) — fetched directly as a single full-res image from `lh3.googleusercontent.com`.
- **Stitching modes** for official Street View panoramas:
  - *Optimized (Default)* — fast, skips empty tile space.
  - *Strict 2:1 Sphere* — pads missing tiles to a full 32×16 grid to prevent morphing/stretching.
  - **"Open in Maps" fallback** — for user photos that need a signed `AF...` token, the app opens Google Maps for you, and you paste the resulting URL back in (Step 2) so the token can be extracted.
- **Metadata injection** — embeds Photo Sphere XMP and binary EXIF GPS coordinates into the downloaded image.
- **Live progress UI** — animated tile grid showing per-tile fetch status (loading / done / failed) plus an overall progress bar.
- **Coordinate-only fallback** — if only lat/lng can be parsed from a link, the app offers a Google Maps link to open Street View at that location.

## Getting Started

Because the app fetches images cross-origin, opening `index.html` directly via `file://` may hit CORS restrictions in some browsers. Serve it locally instead:

```bash
npx serve .
```

Then open the printed local URL (e.g. `http://localhost:3000`) in your browser.

> If fetches still fail due to CORS, a browser extension such as *CORS Unblock* can help.

## Usage

1. **Paste a URL** — copy a link from Google Maps, Street View, or the Google Earth app and paste it into the Step 1 field.
2. The app shows a badge identifying what it found:
   - 🔵 **Official Street View** pano ID
   - 🟣 **User-uploaded 360°** pano ID
   - 🟡 **Coordinates only** (no pano ID) — use the "Open in Maps" link to load Street View, then copy the resulting URL
3. For official Street View panoramas, choose a **Stitching Mode** (Optimized or Strict 2:1 Sphere).
4. Click **Download Panorama**. Progress is shown live as tiles are fetched and stitched.
5. If a user photo requires a signed token, use **Open in Maps**, wait for Street View to load, copy the address bar URL, and paste it into the **Step 2** field to complete the download.

## Supported Link Formats

| Source | Example pattern | Notes |
|---|---|---|
| Google Maps Street View | `.../maps/@lat,lng,3a,...!1s<panoId>!2e0` | Pano ID extracted from `!1s` |
| `panoid=` query param | `?panoid=<panoId>` | Direct match |
| User photo path | `/p/<panoId>` | Treated as user-uploaded |
| Embedded lh3 image URL | `!6shttps://lh3.googleusercontent.com/...` | Direct full-res image link |
| Google Earth app links | `earth.app.goo.gl/...` | Decoded from an embedded protobuf blob |
| Direct pano ID | `CAoS...`, `CI...`, `AF1Q...` | Pasted as-is |

## How It Works (Technical Overview)

- **URL parsing** — a small hand-rolled protobuf decoder (`parseProto`/`readVarint`) walks base64-encoded `data=` blobs from Google Earth links to locate an embedded pano ID string.
- **Tile fetching** — for official Street View, tiles are requested at increasing zoom levels (up to a 26×13 grid at max zoom) and drawn onto a `<canvas>` to reconstruct the full panorama.
- **Direct image fetch** — user-uploaded photos skip tiling entirely; the app strips size/viewport parameters from the `lh3.googleusercontent.com` URL and requests it at `w16000-h8000`.
- **Metadata embedding** — `buildBinaryExifGPS()` constructs a binary EXIF APP1 segment with GPS IFD tags, and Photo Sphere XMP metadata is injected so the resulting JPEG is recognized by 360° photo viewers.

## Tech Stack

Pure HTML/CSS/JavaScript — no frameworks, no dependencies, no build tooling. Fonts are loaded from Google Fonts (`Space Mono`, `Syne`).

## Disclaimer

This tool is intended for personal archival of panoramas you have the right to access and download (e.g., your own uploaded photos, or publicly viewable Street View imagery). Respect Google's Terms of Service and applicable copyright/privacy laws when using it.

## License

No license specified — add one (e.g. MIT) if you plan to distribute this project.
