# Hauraki Gulf Wind — at-a-glance wind dashboard

A single-page web dashboard showing **live wind** (speed, gust, direction) at eight
points around the Hauraki Gulf. It's designed to be read **at a glance, from a distance** —
e.g. mounted on a screen at a yacht's helm — so it uses big numbers and colour-coded tiles,
with a **2-hour wind-speed trend graph** behind each tile (Strava-elevation style) so you can
see whether the wind is building or dropping.

It's one self-contained `index.html`: **no build step, no dependencies, no server required.**

---

## What's in this folder

| File | Purpose |
|------|---------|
| `index.html` | The entire app — HTML, CSS and JavaScript in one file. |
| `README.md`  | This file — how to run and deploy. |
| `AI-CONTEXT.md` | Full project context, design intent, decision history and the reverse-engineered data source — **hand this to an AI assistant** to continue the project. |

---

## Quick start (run it locally)

**Easiest:** double-click `index.html`. It opens in your browser and starts pulling live
data immediately. Done.

**Optional** — serve it over a local web server if you prefer a real URL:

```bash
cd hauraki-gulf-wind
python3 -m http.server 8137
# then open http://localhost:8137 in your browser
```

You need an internet connection (it fetches live data from an external API).

---

## Deploy it (put it on a public URL)

Because it's a single static file, you can host it anywhere that serves static files —
there is **no build step**:

- **Netlify (easiest):** go to <https://app.netlify.com/drop> and drag this whole folder
  onto the page. You get an instant HTTPS URL.
- **Cloudflare Pages / Vercel:** create a new project, point it at this folder,
  leave the build command empty, set the output/publish directory to this folder.
- **GitHub Pages:** push these files to a repo and enable Pages on the branch root.
  URL looks like `https://<you>.github.io/<repo>/`.
- **Any web host:** just upload `index.html`.

All of these serve over **HTTPS**, which the data source requires.

---

## How it works

- Wind data comes from **Zephyr** (<https://www.zephyrapp.nz>), a New Zealand live
  wind-station aggregator, via its public JSON API at `https://api.zephyrapp.nz`.
- On load — and then every 60 seconds — the page calls:

  ```
  GET https://api.zephyrapp.nz/stations
  ```

  which returns ~480 stations, each with its current readings. The page filters that list
  down to the eight stations it shows, matched by station `_id`.
- The API reports wind in **km/h**; the page converts to **knots** (÷ 1.852).
- The API sends permissive **CORS** headers, so the browser can call it directly —
  that's why no backend or proxy is needed.
- For the **2-hour trend graph** behind each tile, the page also calls
  `GET https://api.zephyrapp.nz/stations/{id}/data` (per station, every 5 minutes). That
  endpoint returns ~24 h of 10-minute readings; the page keeps only the last 2 h.

Useful fields on each station object:

| Field | Meaning |
|-------|---------|
| `currentAverage` | average wind speed (km/h) |
| `currentGust`    | gust speed (km/h) — `null` on some buoys |
| `currentBearing` | wind direction in degrees, the direction it is blowing **from** |
| `lastUpdate`     | ISO timestamp of the reading (drives the per-tile age label) |
| `isOffline`      | station currently offline |
| `location.coordinates` | `[lon, lat]` |

---

## The stations shown

Edit these in the `STATIONS` array near the top of the `<script>` block in `index.html`.

| Tile name | Zephyr `_id` | Notes |
|-----------|--------------|-------|
| Passage Rock      | `6631d5ddcf26372d5b80413b` | |
| Tāmaki Strait     | `68f65b7ce3323e552ce3b2a5` | Stands in for **Bastion Point** (not in Zephyr). Buoy → no gust. |
| Tiritiri Matangi  | `69f7dd013fd033f10b5f8696` | |
| Channel Island    | `6631d5ddcf26372d5b8040a2` | |
| Te Kouma Heads    | `6631d5ddcf26372d5b80414c` | |
| Manukau Head      | `6631d5ddcf26372d5b804176` | |
| Rangitoto Buoy    | `68f65ba0e3323e552ce3b2dc` | Buoy → no gust. |
| Whangaparāoa      | `6631d5ddcf26372d5b8040c0` | |

### To change or add a station
1. Open <https://api.zephyrapp.nz/stations> in a browser (it's plain JSON — use a JSON
   viewer or your browser's search to find a station by `name`).
2. Copy that station's `_id`.
3. Add/edit an entry in the `STATIONS` array in `index.html`:
   ```js
   { id:"PASTE_THE_ID_HERE", name:"Your label" },
   ```
4. Save and reload. The layout auto-flows however many stations you list.

---

## Customising

- **Polling rate:** `setInterval(refreshAll, 60*1000)` — change `60` to taste. Note: most
  of these stations only update upstream every ~10 minutes, so polling faster won't get
  you fresher numbers.
- **Wind-strength colour bands:** `bandKey()` and `bandLabel()` define the knot thresholds
  — Light `< 11`, Moderate `< 21`, Fresh `< 31`, Strong `< 34`, Gale `≥ 34`. The colours
  live as CSS variables (`--calm`, `--mod`, `--fresh`, `--gale`) inside the
  `[data-theme="day"]` / `[data-theme="night"]` blocks.
- **Day / Night themes:** the page auto-picks by time of day and you can toggle with the
  ☀ / 🌙 button (the choice is remembered in `localStorage`). Each theme is just a set of
  CSS-variable values.
- **Fullscreen:** the ⛶ button toggles fullscreen — handy for a mounted/kiosk screen.
- **Trend graph:** the background chart shows the last **2 hours** and refreshes every 5 min.
  Adjust the window in `twoHourPoints()` and the cadence in `setInterval(fetchHistory, …)`;
  styling (line/fill opacity, colour) is in `drawGraph()`.
- **Layout:** a CSS grid that fills the viewport — 4 columns on a wide monitor, 2 on a
  tablet, 1 on a phone. Numbers scale with the screen via CSS `clamp()`.

---

## Known limitations / things to be aware of

- **Unofficial API.** Zephyr's API isn't a documented public product — it could change
  without notice. If readings stop, first check that <https://api.zephyrapp.nz/stations>
  still returns JSON and that the `_id`s above still exist.
- **Bastion Point isn't in Zephyr's network**, so the board uses the **Tāmaki Strait Buoy**
  in that slot.
- **Buoy stations** (Tāmaki Strait, Rangitoto) report an average only — **no gust**
  (shown as `G –`) — and update less frequently.
- **Not official Coastguard / MetService data.** These are similar station feeds, but treat
  the dashboard as situational awareness, **not** an official forecast or observation
  source. Always cross-check Coastguard / MetService for any safety-critical decision.

---

## Ideas for continuing

- A **"strongest wind first"** sort toggle (currently a fixed geographic order).
- A **red-tinted night mode** to preserve night vision on passage.
- Cache the last good readings in `localStorage` so the board still shows something if the
  connection drops.

---

*Built as a single static page — no frameworks, no tooling. Open it, edit it, ship it.*
