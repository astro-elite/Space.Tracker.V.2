# SPACE TRACKER

**KOINOTNI OʻRGANING. NOMAʼLUMNI KASHF ETING.**
*Developed by Noven Studio — Building the Future Through Technology.*

SPACE TRACKER is a scientific astronomy platform. The whole website is one file, `index.html`, served by GitHub Pages. A small Cloudflare Worker, `worker.js`, keeps API keys secret and proxies the services that do not allow browser requests.

| File | Purpose |
|---|---|
| `index.html` | The entire website: HTML, CSS, JS and the UZ / EN / RU dictionaries. |
| `assets/noven-studio-logo.png` | A 512 px web copy of the Noven Studio logo, with transparency kept. It is cropped and padded to a square and not otherwise changed. |
| `Noven studio/Metallic Silver Angular Emblem.png` | The original logo file (1263 × 1246, RGBA). |
| `worker.js` | Cloudflare Worker: NASA key injection, JPL Horizons, original-resolution NASA image downloads, NASA article text, translation and Space Intelligence (Gemini). |
| `wrangler.toml`, `package.json` | Worker deployment config. The Worker bundles `@anthropic-ai/sdk`. |

## 1. Deploy the website (GitHub Pages)

1. Commit `index.html` and the `assets/` folder to the repository root.
2. In **Settings → Pages**, deploy from branch `main`, folder `/ (root)`.
3. The site works without the Worker. In that mode:
   - NASA calls use NASA's public `DEMO_KEY`, which is heavily rate-limited.
   - Horizons and spacecraft positions fall back to computed Keplerian positions; translation shows the original text with a small "original" tag; Space Intelligence says it is unavailable. Nothing shows placeholder data, and visitors never see backend configuration details.
   - Gallery "Download image" opens the original NASA file in a new tab (the NASA asset host sends no CORS headers, so the page cannot save it directly without the Worker).

## 2. Deploy the Worker (Cloudflare)

```bash
npm install                      # installs @anthropic-ai/sdk + wrangler (needed: worker.js imports the SDK)
npx wrangler login
npx wrangler deploy                                # creates the TR_KV namespace on the first run
npx wrangler secret bulk .dev.vars                 # uploads the keys from the local, git-ignored .dev.vars
```

`.dev.vars` (never commit it) holds one `NAME="value"` per line:

| Name | Purpose |
|---|---|
| `GEMINI_API_KEYS` | Space Intelligence and translation. One key or several, comma-separated; a rejected or exhausted key is skipped. |
| `NASA_API_KEY` | Optional, https://api.nasa.gov/. Without it the Worker uses NASA's shared `DEMO_KEY`. |
| `GOOGLE_TRANSLATE_API_KEY` | Optional. When set, Google Cloud Translation is used instead of Gemini for translation. |
| `ANTHROPIC_API_KEY` | Optional. Space Intelligence uses Claude only when no Gemini key is set. |

Model order can be changed with the variables `GEMINI_MODELS` and `GEMINI_TRANSLATE_MODELS`.

- `ALLOWED_ORIGINS` in `wrangler.toml` is set to `https://astro-elite.github.io`, the origin only, without the path. Change it if the site moves.
- After deploying, open `https://<worker>.workers.dev/health`. It reports which keys are configured, never their values.
- Then set `CONFIG.PROXY_URL` near the top of the `<script>` in `index.html` to the Worker URL, and commit.
- Optional extras:
  - Global visitor counter: create a KV namespace (`COUNTER`) and uncomment its block.
  - Shared rate limiting: uncomment the `[[ratelimits]]` block.

### Worker routes

| Route | Upstream | Notes |
|---|---|---|
| `GET /nasa/<path>` | api.nasa.gov | Allow-listed paths only (APOD, NeoWs, DONKI); the key is added server-side. |
| `GET /jpl/cad.api`, `/jpl/sbdb.api` | ssd-api.jpl.nasa.gov | No CORS upstream. |
| `GET /exo?query=` | Exoplanet Archive TAP | Read-only `SELECT` on `ps` / `pscomppars`. |
| `GET /horizons[?date=YYYY-MM-DD]` | JPL Horizons | Vectors for the 8 planets. Without `date`: the current hour. With `date`: 1800–2050, 00:00 UTC. |
| `GET /bodies` | JPL Horizons | Spacecraft positions; see the list below. |
| `GET /image?url=` | images-assets.nasa.gov | Gallery downloads. Listed origins only; only `https://images-assets.nasa.gov/image/<id>/<file>.(jpg\|png\|gif\|webp\|tif)`; no redirects followed; ≤ 250 MB; 20 requests/min per IP. Not a general proxy. |
| `POST /translate`, `POST /api/translate` | Gemini, or Google Cloud Translation v2 | Listed origins only; cached per string for 30 days (edge cache + `TR_KV`). Protected names and numbers are checked; a string that fails stays in the original language. |
| `GET /article?url=` | www.nasa.gov, science.nasa.gov | Listed origins only; plain paragraphs of NASA's own articles (APOD excluded); no redirects followed. |
| `POST /ai`, `POST /api/ai` | Gemini (or Claude, see above) | Listed origins only; 6 requests per minute per IP per isolate; identical first questions are cached for 5 minutes. |
| `GET/POST /counter` | Worker KV | Optional visitor counter. |
| `GET /health` | — | Shows whether each key is configured. |

`/bodies` covers Voyager 1 and 2, New Horizons, Parker Solar Probe, JWST, Europa Clipper, Psyche and Lucy. It returns positions relative to the Sun and to Earth for the current hour. The Horizons IDs were checked against the live API. These are JPL trajectory solutions, not live telemetry, and the site labels them that way.

**Security model**

- No key is ever in `index.html`.
- CORS answers only `ALLOWED_ORIGINS`. `/translate`, `/article` and `/ai` refuse any origin that is not explicitly listed, so the Worker cannot be used as an open proxy.
- Requests are size-limited and validated.
- Upstream error bodies are not echoed back.

**Space Intelligence**

- The Worker runs the model with server-side tools, so answers are grounded in real data. Each tool calls one fixed upstream with validated parameters:
  - ISS position (wheretheiss.at)
  - body and spacecraft distance (JPL Horizons)
  - upcoming launches (LL2)
  - space weather (NOAA SWPC)
  - recent exoplanets (Exoplanet Archive)
  - news search (SNAPI)
  - NASA image search
- The page also sends the Moon phase and the next events it computed itself, labelled as site-computed.
- The citations shown are only the sources the tools actually used.
- Gemini models are tried in order (`GEMINI_AI_MODELS` in `worker.js`), so an overloaded model or an exhausted key does not stop the assistant.
- Without a key, the page shows a setup message, never a placeholder answer.

## 3. Sections

The navigation was simplified: Space Weather, Asteroid Watch, Exoplanet Explorer, Space Events and Space Data Center were removed as pages. The astronomy-event engine is kept internally because Space Intelligence uses it as site context, and the Worker's JPL / Exoplanet / NOAA integrations stay available to the AI tools. Old links such as `#weather` fall back to the dashboard.

| # | Section | Data |
|---|---|---|
| 01 | Dashboard | Tashkent and UTC clocks, ISS, APOD, launches, Moon, featured missions, news (opens the internal reader), quick access |
| 02 | ISS Live Tracker | wheretheiss.at, polled every 5 s; Three.js globe with clouds, city lights, terminator and ground track |
| 03 | Solar System *(+ Moon Phase)* | 3D scene; JPL Keplerian elements anchored to Horizons; NASA fact-sheet properties; Meeus Moon engine |
| 04 | Deep Space Explorer | Curated catalogue of 45 objects, live NASA images, JWST / Hubble / Chandra feeds |
| 05 | Mission Control *(+ Mars Explorer)* | 34 missions; live Horizons positions |
| 06 | Space Gallery | NASA Image and Video Library; viewer with favourite, original-resolution download and full screen |
| 07 | Launch Center | Launch Library 2 |
| 08 | Space News | Spaceflight News API; **internal article reader** at `#news/<id>` |
| 09 | Solar System Timeline | Any date 1800–2049, date comparison, "Verify with JPL Horizons" |
| 10 | Space Missions Map | Live ISS, spaceports, launch pads, night side, deep-space spacecraft |
| 11 | Space Intelligence | Gemini via the Worker |
| 12 | Astronomy Quiz | Local questions over 3 levels; works offline |
| 13 | About | Project, Noven Studio, scientific integrity principles, legal / non-endorsement notice |
| ★ | Favorites | localStorage; no account needed |

### Gallery viewer

- ✕, Escape and a click on the backdrop close it; clicks on the image or inside the dialog do not. (The old ✕ failed because the dialog content was stacked above the button and swallowed most clicks.)
- Closing restores page scrolling, the scroll position and focus on the tile that opened it, and clears the dialog content.
- **Download image** reads the item's asset manifest (`images-api.nasa.gov/asset/<nasa_id>`), picks `~orig` (then `~large`, `~medium`, `~small`; never an upscaled thumbnail) and saves it through the Worker's `/image` route as `<title>_<nasa_id>.<ext>`. One request per image at a time, with a busy state and error messages.

### News reader

NASA's own articles (public domain in the US) are shown in full through the Worker's `/article` route. For every other outlet the reader shows only the summary provided by the Spaceflight News API; third-party articles are not copied. One source line links to the original page.

These systems run across the whole site:

- **Global search:** press `/` or Ctrl+K.
- **Favorites:** stored in localStorage.
- **Languages:** UZ (default), EN and RU. Switching does not reload the page. External text is translated through the Worker (Gemini). Launch vocabulary, object, mission and spaceport names are localized in the page itself. Failed translations show the original text with a marker.
- **Navigation:** fixed sidebar on desktop (> 1024 px); hamburger + slide-in drawer on tablets and phones (closes on ✕, scrim, Escape or any link).
- **Data statuses:** LIVE, UPDATED, STALE and OFFLINE.
- **Validation layer:** `VALID`.
- **Passive API health monitor:** it records the requests the page makes anyway and polls nothing.

## 4. Branding and links

- The public name is **SPACE TRACKER** everywhere (title, loader, header, About, footer) — no version suffix.
- The original Noven Studio logo (`assets/noven-studio-logo.png`, from `Noven studio/`) appears in the About studio card, the footer, the sidebar credit and, subtly, on the loading screen.
- Footer: SPACE TRACKER · logo · "Developed by Noven Studio" · "Building the Future Through Technology." · Telegram channel button · copyright · one-line image/texture credit. Translated into UZ / EN / RU.
- The **Kanalga obuna boʻling** button opens `https://t.me/Koinot_Rasmlari_Va_Videolari` in a new tab with `rel="noopener noreferrer"`.
- No personal developer attribution is shown. The visitor counter was removed from the footer (the Worker's optional `/counter` route still exists).

## 5. Known limitations

- **Satellites other than the ISS** are not shown, because no verified CORS-enabled live provider is integrated. The map says so.
- **Solar eclipses:** local circumstances for Uzbekistan are not computed; each one links to NASA. Lunar-eclipse visibility is computed from the Moon's altitude at Tashkent at the syzygy instant, which is close to maximum.
- **Meteor-shower peaks** use IMO solar longitudes and are accurate to within a few hours. Weather is never considered.
- **Mission statuses** are curated and reviewed on the date shown on the page. Re-check them with each mission's official link.
- **Without the Worker:** NASA `DEMO_KEY` limits are low (APOD often returns HTTP 429), and Horizons, spacecraft positions, translation, image downloads and AI are unavailable or reduced.
- **News reader:** outside NASA, only the outlet's summary is shown (copying the full article would infringe the publisher's rights).
- **Gemini free tier:** daily quotas are limited. When they run out, new texts stay in the original language with a marker until the quota resets; Google may use free-tier content to improve its products.
- **Downloads:** some NASA items only have a large TIFF as `~orig`; that file is downloaded as-is (it can be big).
- **WebGL:** if it is unavailable, the 3D scenes show a message; all other sections still work.
