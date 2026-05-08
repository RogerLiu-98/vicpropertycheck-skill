# vicpropertycheck-skill

Ask Claude anything about a Victorian (Australia) address.

A [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skill that pulls
authoritative property data — planning zones, bushfire risk, suburb crime, housing stock,
nearby facilities — and answers questions in plain language.

## What it does

- **Planning** — zones and overlays from VicMap (heritage, design, vegetation, parking, etc.)
- **Bushfire** — Bushfire Management Overlay (BMO) and Bushfire Prone Area status
- **Crime** — suburb-level rates and per-category breakdown from OpenStats
- **Housing** — owner-occupied / private-rental / public-housing share for the suburb
- **Facilities** — schools, public transport, shops within walking distance
- **Summarize** — runs all of the above for one address in a single call

## Demo

> **You**: What's the bushfire risk and zoning for 1 Spring St, Melbourne?
>
> **Claude**: 1 Spring St is in zone PUZ7 (Public Use Zone — Other Public Use), with two
> overlays: PO12 (Parking Overlay, Precinct 12) and HO175 (Heritage Overlay). Bushfire
> risk is negligible — the address is well outside Victoria's mapped Bushfire Prone Areas
> and not subject to the BMO.

Claude detects the trigger, runs `vic-property summarize`, and reads the JSON back.

## Install

Two pieces — the CLI (Python package) and this skill (the prompt that drives it).

```bash
# 1. Install the CLI
pipx install vicpropertycheck

# 2. Install this skill (Linux / macOS)
git clone https://github.com/RogerLiu-98/vicpropertycheck-skill ~/.claude/skills/vic-property-check
```

```pwsh
# Windows (PowerShell)
git clone https://github.com/RogerLiu-98/vicpropertycheck-skill "$env:USERPROFILE\.claude\skills\vic-property-check"
```

That's it for planning, bushfire, crime, and housing — they use free VIC government APIs.

For address lookups and nearby facilities (which call Google Maps), you'll need either
your own Google Maps API key or a free VicPropertyCheck account. See
[INSTALL.md](./INSTALL.md) for the full walkthrough.

## Data sources

| Source | Used for | Auth |
|---|---|---|
| [VicMap Planning](https://mapshare.vic.gov.au/) | zones, overlays | none |
| [VicMap Bushfire](https://mapshare.vic.gov.au/) | BMO, prone areas | none |
| [OpenStats](https://www.openstats.com.au/) | crime, housing | none |
| [Google Maps](https://developers.google.com/maps) | geocoding, places | API key OR proxy login |

## Limitations

- **Victoria only.** Coordinates are bounds-checked; addresses elsewhere are rejected.
- **No valuations.** Sale prices and rental yields aren't part of this dataset.
- **Cached.** Responses are cached for ~7 to 90 days depending on the source. Treat as
  "current within a few weeks," not real-time.

## Links

- CLI package on PyPI: https://pypi.org/project/vicpropertycheck/
- CLI source: https://github.com/RogerLiu-98/vic-property-due-diligence-mcp
- VicPropertyCheck web app: https://vicpropertycheck.com.au

## License

MIT — see [LICENSE](./LICENSE).
