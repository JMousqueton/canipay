# 💰 Can I Pay Ransomware?

A static, single-page interactive world map showing whether paying a ransomware demand is legal in a given country — along with reporting obligations, sanctions exposure, and official citations for each jurisdiction.

Live concept: click (or hover) a country to see its payment-legality status, a plain-English summary, and links to the underlying legal sources. The United States additionally drills down into a state-by-state breakdown.

Maintained by [ransomware.live](https://www.ransomware.live) as a community resource. Original concept based on [rkovar/ransomwarelegality](https://github.com/rkovar/ransomwarelegality).

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire application — markup, CSS, and JS in one file. No build step. |
| `countries.json` | Legal-status dataset, keyed by ISO 3166-1 alpha-3 country code. Fetched client-side at load time. |

There is no backend, build tooling, or package manager — this is deployed as static files behind nginx/whatever serves the rest of ransomware.live.

## How it works

`index.html` renders a dark [Leaflet](https://leafletjs.com/) map (CartoDB dark, no-labels basemap) and, on load, fetches:

- `countries.json` (same directory) — the legality dataset.
- A world GeoJSON from `cdn.jsdelivr.net/gh/holtzy/D3-graph-gallery` — country polygons.
- A US states GeoJSON from `cdn.jsdelivr.net/gh/PublicaMundi/MappingAPI` — used to overlay per-state detail on top of the single USA polygon (the USA polygon itself is hidden once the state layer loads).

Each country polygon is matched to a `countries.json` entry by ISO alpha-3 code (`feature.id` / `ISO_A3` / `iso_a3` / `ADM0_A3`, tried in order — see `getCountryEntry()`), colored by status, and made interactive:

- **Hover** → floating tooltip (`showTip()`) with name, status, short info, and source links.
- **Click** → right-hand info panel (`showPanel()`) with the full write-up, citations, and (for the USA) the state-by-state table.

An "About" modal shown on first visit (cookie-gated: `canipay_seen`) carries the legal disclaimer.

Matomo analytics (`stats.mousqueton.io`, site ID `10`) is wired in for page-view tracking.

## `countries.json` schema

Top-level object keyed by ISO 3166-1 **alpha-3** country code:

```jsonc
{
  "USA": {
    "name": "United States",
    "status": "complicated",   // see status values below
    "info": "Free-text summary shown in the tooltip and info panel.",
    "citations": [
      { "label": "Source name shown in the UI", "url": "https://..." }
    ],
    // Optional — currently only present for USA:
    "stateDetails": [
      { "name": "Florida", "status": "illegal", "note": "for state agencies & municipalities" }
    ]
  }
}
```

### Status values

Defined in the `STATUS` map in `index.html`:

| Status | Meaning | Color |
|---|---|---|
| `legal` | Legal to pay | green `#22c55e` |
| `report` | Legal, but payment/incident must be reported | purple `#a855f7` |
| `complicated` | Mixed / conditional / in flux | yellow `#eab308` |
| `illegal` | Illegal to pay | red `#ef4444` (currently only used at the US state level, via `stateDetails`) |
| `unknown` / omitted | No entry, or a status not in the map above | falls back to gray "Unknown / no data" styling |

A country with no key in `countries.json` (or an unrecognized `status` value) automatically renders as "Unknown / no data" — no special-casing needed when adding partial data.

## Adding or updating a country

1. Add/edit an entry in `countries.json` under its ISO alpha-3 code (must match what the GeoJSON source reports as `id`/`ISO_A3`).
2. Keep `info` factual and cite primary or reputable secondary sources (government guidance, major law firms, etc.) in `citations`.
3. Pick the narrowest accurate `status` — prefer `report`/`complicated` over `legal` when reporting or sanctions obligations materially change the picture.
4. For the USA only, `stateDetails` entries are matched to the states GeoJSON by exact `name` (e.g. `"Florida"`, `"D.C."`); any US state not listed falls back to `complicated` / "Federal OFAC restrictions apply" (see `getStateInfo()`).
5. Validate the JSON (e.g. `python3 -m json.tool countries.json > /dev/null`) and spot-check new citation URLs before committing — dead links silently degrade the source list in the UI.

## Disclaimer

This is not legal advice, and the dataset explicitly excludes analysis of payments to sanctioned/designated entities (see the in-app "About" modal). Laws in this space change quickly — treat `countries.json` as a snapshot that needs periodic re-verification against current legislation, not a source of truth to build automation on.

## Local development

No build step required — serve the directory with any static file server and open it in a browser, e.g.:

```bash
git clone https://github.com/jmousqueton/canipay
cd canipay
python3 -m http.server 8000
# then open http://localhost:8000
```
