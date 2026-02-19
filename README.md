# Earth360 Downloader

A single-file, client-side tool for downloading full-resolution Google Street View and user-uploaded 360° panoramas with proper **Photo Sphere XMP metadata** — so they open in panorama mode in any gallery app.

**[Try it live (vercel) ](https://google-pano-downloader.vercel.app/)**

**[Try it live (github pages)](https://jasonbornxx.github.io/Google-pano-downloader/)**

---

## Why open source?

Tools like this exist behind paywalls or as sketchy browser extensions. But the underlying technique — tile stitching, protobuf decoding, XMP injection — is just clever glue between endpoints that your browser already uses every time you open Google Maps. There is no secret sauce worth hiding, and keeping it open means anyone can audit exactly what it does with their data (spoiler: nothing leaves your device).

Selling downloads of Google's imagery would also violate their Terms of Service. Open source keeps this honest — a personal utility, not a product.

---

## What it does

- Accepts any Google Maps, Street View, or `earth.app.goo.gl` link
- Automatically detects whether it is an **official Street View** panorama or a **user-uploaded 360° photo**
- Downloads the full-resolution image and saves it as a JPEG with Photo Sphere XMP metadata embedded
- Gallery apps (Google Photos, Samsung Gallery, Windows Photos, iOS Photos) recognise the file as a 360° panorama and open it in panorama/VR mode automatically

---

## How it works

### Official Street View panoramas

Google stores Street View imagery as a grid of 512x512px tiles served from `cbk0.google.com`. The app fetches all tiles concurrently, draws them onto an HTML5 canvas to stitch the full equirectangular image, converts to JPEG, then injects a Photo Sphere XMP metadata block before saving.

| Zoom | Grid | Output resolution |
|------|------|-------------------|
| Z1 | 2x1 | ~1K |
| Z2 | 4x2 | ~2K |
| Z3 | 8x4 | ~4K |
| Z4 | 16x8 | ~8K |
| Z5 | 26x13 | ~13K x 6.6K |

### User-uploaded 360 photos (CI... pano IDs)

These are not tiled — Google serves them as a single pre-stitched JPEG from `lh3.googleusercontent.com`. The important detail is getting the URL right:

Maps URLs often embed viewport-cropped versions with camera parameters like `-pi49.79-ya332.47-fo100` that instruct Google to serve a perspective crop (a flat screenshot of one direction) instead of the full sphere. The app strips all viewport parameters and requests `=w16000-h8000-k-no` to get the raw equirectangular source. If the original already contains `GPano` XMP (most photographer uploads do), it is preserved exactly as downloaded.

### Google Earth app links (earth.app.goo.gl)

Earth share links encode the panorama data as a base64-encoded protobuf binary inside the `data=` URL parameter. The app includes a minimal protobuf decoder that recursively walks the binary structure to extract the pano ID, coordinates, and heading angle — all without any server or external library.

For user-uploaded panos from Earth links, the signed `AF...` token required to fetch the image is not present in the Earth link itself. It only appears after a real browser session loads Google Maps. The app handles this with a two-step flow:

1. Constructs the equivalent Maps Street View URL from the extracted pano ID and coordinates, then opens it in a new tab
2. After Maps loads, the user copies the URL from the address bar and pastes it back into the app
3. The app extracts the `!6s` token, strips viewport params, and fetches the full-res image

### Photo Sphere XMP injection

For stitched Street View images, XMP metadata is written directly into the JPEG binary. The app inserts an `APP1` marker block immediately after the JPEG `SOI` header containing the `GPano` namespace:

```
GPano:ProjectionType       = equirectangular
GPano:FullPanoWidthPixels  = {width}
GPano:FullPanoHeightPixels = {height}
GPano:UsePanoramaViewer    = True
GPano:CroppedAreaLeftPixels = 0
GPano:CroppedAreaTopPixels  = 0
```

This is the same metadata standard used by Google Street View camera rigs, Ricoh Theta, Insta360, and other 360 cameras. Any app that supports Photo Sphere (which is most modern gallery apps) will automatically detect the panorama.

---

## Limitations

**CORS** — Tile fetching requires the browser to allow cross-origin image requests from `cbk0.google.com`. If tiles fail, run the file via a local server (`npx serve .`) or use the CORS Unblock browser extension.

**CIHM panos without a token** — User-uploaded panoramas that come from Earth app links require the two-step Maps URL flow because the signed AF token is session-bound and cannot be derived from the pano ID alone.

**Google Terms of Service** — This tool accesses internal Google endpoints that are not part of any public API. It is intended for personal, non-commercial use only. Do not use it to scrape imagery at scale.

---

## Running locally

No build step. No dependencies. No npm install.

```bash
git clone https://github.com/yourname/earth360-downloader
cd earth360-downloader
npx serve .
```

Then open `http://localhost:3000`. Running via a local server rather than opening the file directly avoids CORS issues with tile fetching.

---

## Deploying to Vercel

```bash
vercel deploy
```

It is a single HTML file. Vercel serves it as a static site with zero configuration needed.

---

## Contributing

PRs welcome. Areas that would make this meaningfully better:

**Browser extension port** — would eliminate the CORS wall and the manual Maps step entirely, making user-uploaded pano downloads fully automatic in one click.

**Batch mode** — accept multiple URLs and queue downloads.

**Heading preservation in XMP** — the initial view direction from the original link could be written into `GPano:InitialViewHeadingDegrees` so the panorama opens facing the right direction instead of defaulting to north.

**Better protobuf decoding** — the current decoder is hand-rolled and minimal. A proper proto3 implementation would handle more edge cases.

---

## Legal

Released under the MIT License.

The imagery downloaded using this tool belongs to Google and/or the individual photographers who contributed it and is subject to [Google's Terms of Service](https://www.google.com/intl/en/policies/terms/). This tool is intended for personal, offline, non-commercial use only. The authors take no responsibility for misuse.
