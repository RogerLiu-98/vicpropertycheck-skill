# Installing the vic-property-check skill

The skill is a thin Claude Code wrapper around the `vic-property` Python CLI. Both pieces
need to be on disk for it to work.

## 1. Install the CLI

The skill shells out to a `vic-property` CLI that ships in the `vicpropertycheck` Python
package on PyPI.

```bash
pipx install vicpropertycheck
```

Verify:
```bash
vic-property --help
```
Should print the subcommand list (`resolve-address`, `planning`, `bushfire`, `housing`,
`crime`, `facilities`, `summarize`).

## 2. Choose how to call Google Maps

Address geocoding (`resolve-address`, `summarize`) and nearby-facilities lookup need
to call Google Maps. Planning, bushfire, crime, and housing don't — they hit free VIC
government and OpenStats endpoints, so you can skip this step entirely if those four
are all you need.

You have two paths:

### Option A — bring your own Google Maps key (self-hosted)

Most direct. You pay Google directly; nothing routes through VicPropertyCheck.

1. https://console.cloud.google.com/apis/credentials → create an API key.
2. Enable the **Geocoding API** and **Places API (New)** for the project.
3. Restrict the key to those two APIs.
4. Make the key visible to the CLI. Two ways:
   ```bash
   # Option 1 — env var (per-shell):
   export GOOGLE_MAPS_API_KEY=AIza...        # bash/zsh
   $env:GOOGLE_MAPS_API_KEY = "AIza..."      # pwsh

   # Option 2 — persistent .env file (any shell, any directory):
   # POSIX:    ~/.config/vicpropertycheck/.env
   # Windows:  %APPDATA%\vicpropertycheck\.env
   # File contents (one line):
   #   VIC_PROPERTY_API__GOOGLE_MAPS_API_KEY=AIza...
   ```
   Either form works. The .env file is preferred for `pipx`-installed CLIs
   since it doesn't need the env var set in every new shell.

### Option B — sign in via VicPropertyCheck (hosted proxy)

No Google Cloud setup. Calls go through the VicPropertyCheck web app, which uses its
own server-side key. You authenticate once with Firebase.

1. Open https://vicpropertycheck.com.au/skill-auth and sign in.
2. Click **Generate credentials**. Copy the JSON blob shown.
3. Pass it to `vic-property login`:
   ```bash
   # paste from clipboard, then Ctrl-D (Linux/macOS) or Ctrl-Z Enter (Windows)
   vic-property login

   # or pipe in directly
   pbpaste | vic-property login                          # macOS
   xclip -selection clipboard -o | vic-property login    # Linux
   Get-Clipboard | & vic-property login                  # pwsh
   ```
4. The credentials are saved to `~/.config/vicpropertycheck/credentials.json` (mode 0600
   on POSIX). They don't expire — log in once.

If both options are configured, `GOOGLE_MAPS_API_KEY` wins (the self-hosted path is
preferred so you keep your usage off our quota).

## 3. Install the skill

Clone this repository directly into your Claude Code skills directory:

```bash
# Linux / macOS
mkdir -p ~/.claude/skills
git clone https://github.com/RogerLiu-98/vicpropertycheck-skill ~/.claude/skills/vic-property-check
```

```pwsh
# Windows (PowerShell)
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
git clone https://github.com/RogerLiu-98/vicpropertycheck-skill "$env:USERPROFILE\.claude\skills\vic-property-check"
```

To update later, `cd` into that directory and run `git pull`.

## 4. Verify end-to-end

In a fresh Claude Code session, ask something like:

> What's the bushfire risk and zoning for 1 Spring St, Melbourne?

Claude should detect the trigger, run `vic-property summarize ...`, and answer from the
JSON. If it doesn't, run the CLI manually first to confirm both pieces are wired up:

```bash
vic-property summarize --address "1 Spring St, Melbourne VIC 3000" --pretty
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `command not found: vic-property` | CLI not installed, or pipx bin dir not on PATH (`pipx ensurepath`). |
| `Configuration error: Google Maps API key is required` | Env var unset, or set in the wrong shell. |
| `Address could not be geocoded` | The address resolved to nothing — usually a typo or a non-VIC address. |
| Stale data | The CLI caches results in a SQLite file. Delete `~/.local/share/vicpropertycheck/cache.db` (Linux/macOS) or `%LOCALAPPDATA%\vicpropertycheck\cache.db` (Windows) to force a refresh. |
