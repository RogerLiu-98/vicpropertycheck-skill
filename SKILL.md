---
name: vic-property-check
version: 0.1.2
description: |
  Look up Victorian (Australia) property data from authoritative sources: planning zones &
  overlays (VicMap), bushfire management overlays, suburb crime statistics, suburb housing
  stock breakdown, and nearby facilities (schools, transport, shops). Backed by the
  vic-property CLI which calls VicMap ArcGIS, OpenStats, and Google Maps APIs. Use when the
  user asks anything about a specific Victorian address — zoning, fire risk, neighbourhood
  crime/housing context, what's nearby — or wants a one-shot summary report.
allowed-tools:
  - Bash
  - Read
---

# vic-property-check

Use this skill whenever the user asks a question about a **Victorian (Australian) property
or suburb** that touches:

- planning zones / overlays (zoning, heritage, design, vegetation)
- bushfire risk (BMO, prone areas)
- suburb crime statistics
- suburb housing stock (rental / owner-occupied / public housing)
- nearby facilities (schools, public transport, shops)
- a generic "tell me about this property" / "is this a good place to buy" request

The skill drives the `vic-property` CLI. Each subcommand returns JSON on stdout. Pipe or
read the JSON, then summarise the parts that answer the user's actual question — don't
dump raw JSON unless they ask for it.

## Prerequisites (check once per session)

Run this and confirm both:

```bash
command -v vic-property >/dev/null 2>&1 && echo "CLI OK" || echo "MISSING_CLI"
[ -n "$GOOGLE_MAPS_API_KEY" ] && echo "KEY OK"
[ -f ~/.config/vicpropertycheck/credentials.json ] && echo "PROXY_CREDS OK"
```

- If `MISSING_CLI`: tell the user to run `pipx install vicpropertycheck` and stop. Do not
  invent fallback data.
- If `KEY OK` is printed: the CLI will call Google directly using the user's key.
- Else if `PROXY_CREDS OK` is printed: the CLI will route Google calls through the
  VicPropertyCheck web app. No further setup needed — the user is logged in.
- Else: tell the user they have two options:
  1. **Self-hosted**: `export GOOGLE_MAPS_API_KEY=<their Google Maps key>`.
  2. **Hosted (no key needed)**: visit `https://vicpropertycheck.com.au/skill-auth`,
     sign in, copy the JSON blob, then run `pbpaste | vic-property login` (or
     `clip.exe | vic-property login` on Windows / paste manually).
  Planning, bushfire, crime, and housing work without either path — they hit free
  VIC government APIs. If the user only wants those, just continue.

## Choosing a command

| User question | Command |
|---|---|
| "Tell me about this address" / "Is X a good place to live?" | `summarize` |
| Just the address → coords | `resolve-address` |
| Zoning / overlays | `planning` (needs lat/lon — geocode first) |
| Bushfire risk | `bushfire` (needs lat/lon) |
| Crime in a suburb | `crime` |
| Housing stock | `housing` |
| Schools / transport / shops nearby | `facilities` (needs lat/lon) |

**Default to `summarize`** for any open-ended "tell me about" request. It runs every check
in one go and returns a single JSON blob — no need to chain calls yourself.

## Examples

### One-shot property report
```bash
vic-property summarize --address "1 Spring St, Melbourne VIC 3000" --pretty
```
The output has top-level keys `address`, `planning`, `bushfire`, `housing`, `facilities`,
`crime`. Read the keys the user asked about and answer in plain language.

### Targeted lookups
```bash
vic-property resolve-address --address "1 Spring St, Melbourne"   # → lat, lon, suburb
vic-property planning  --lat -37.8113 --lon 144.9737              # zones + overlays
vic-property bushfire  --lat -37.8113 --lon 144.9737              # BMO + prone areas
vic-property crime     --suburb "Melbourne"                       # per-category stats
vic-property housing   --suburb "Melbourne" --lat -37.81 --lon 144.97
vic-property facilities --lat -37.8113 --lon 144.9737 --radius-m 1500 --max-per-category 3
```

Quote any address or suburb that contains spaces.

## Reading the output

Every command emits **one JSON object on stdout**. Errors:

- `{"error": "...", "code": "NOT_FOUND"}` and exit code 1 → address didn't resolve. Tell
  the user the address wasn't recognised; offer to retry with a more complete form
  (street + suburb + postcode).
- Anything written to stderr or non-zero exit without an `error` JSON → infrastructure
  problem (missing key, network). Surface the stderr text to the user verbatim.

For `summarize` the `housing` and `facilities` blocks may include a `*_error` sub-key when
a single sub-call fails — those are non-fatal. Mention them only if directly relevant.

## What this skill is NOT for

- Property valuations / sale price estimates — we don't have that data.
- Anywhere outside Victoria — coordinates are bounds-checked to VIC. Tell the user.
- Real-time data. Two-stage lag stacks here:
  1. **Source cadence.** The Crime Statistics Agency Victoria publishes quarterly,
     with a 4-6 month lag from the period closing to the data going live. OpenStats
     republishes on roughly the same cycle. VicMap planning + bushfire data updates
     when the gazette does — slow, irregular, rarely within a single year.
  2. **CLI cache.** On top of that, the CLI caches OpenStats responses for up to
     90 days locally and VicMap responses for ~30 days. So a fresh CLI lookup may
     itself be replaying a snapshot from weeks ago.

  When the user asks about freshness or about a specific recent news event:
  - Don't claim the data reflects events from the last few weeks. It almost
    certainly doesn't.
  - Name the upstream source by name (Crime Statistics Agency Victoria for crime,
    OpenStats for housing, VicMap for planning/bushfire) so they know where to
    look for live information.
  - Point the user to crimestatistics.vic.gov.au or VicPol media releases for
    operational reporting that hasn't yet been folded into the statistics.
