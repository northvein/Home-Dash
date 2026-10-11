# Home Dash

## Overview

A self-hosted, wall-mounted family calendar display running on a 10-inch Android tablet in a picture frame. The display is permanently on, showing the current week's events, live UK weather, the time, and upcoming bin collections — all themed in the Weyland-Yutani corporate terminal aesthetic from the Alien franchise: `#F5C518` gold on near-black, sharp corners, all-caps tracking.

Everything runs locally on your home network. There is no cloud dependency, no subscription, and no data leaves your network. Calendar feeds are fetched server-side by a Python script, eliminating browser CORS restrictions entirely.

## Architecture

```
[Google / iCloud / CalDAV]
         │  HTTPS (no CORS — Python, not browser)
         ▼
      sync.py          ← runs on your home server / Pi
         │  writes .ics files to disk
         ▼
      data/ics/
         │  HTTP — same origin, no CORS
         ▼
      serve.py          ← Python HTTP server
         │
         ▼
   home-calendar.html  ← displays on the tablet
```

`sync.py` fetches remote calendar feeds (no browser CORS restrictions). `serve.py` serves the HTML and synced `.ics` files over the local network, and triggers a background sync every 15 minutes. The HTML reads `/ics/cal_1.ics` from the same server it was loaded from — same origin, so no CORS policy applies.

Everything is packaged in a Docker container with a single bind-mounted `/data` volume for persistence.

## Quick start

Requires Docker and Docker Compose. Put all files in one folder.

1. Start the container:

   ```bash
   docker compose up -d
   ```
2. Add your calendar URLs — edit the file created on first run:

   ```bash
   nano data/calendars.json
   ```
3. Trigger an immediate sync:

   ```bash
   docker compose exec home-calendar python3 /app/sync.py
   ```
4. Find your server's local IP:

   ```bash
   hostname -I
   ```
5. On the tablet, open:

   ```
   http://<server-ip>:8080/home-calendar.html
   ```

   Set this as the browser homepage. Enable **Stay awake while charging** in Android developer options, and use a kiosk browser (e.g. Fully Kiosk Browser) for a permanent display.

## File layout

```
project-folder/
├── Dockerfile                  ← image definition
├── docker-compose.yml          ← one-command deployment
├── entrypoint.sh               ← first-run bootstrap
├── serve.py                    ← HTTP server + background sync
├── sync.py                     ← fetches remote .ics feeds
├── home-calendar.html          ← the display (open this on the tablet)
├── calendars.default.json      ← template config (copied to /data on first run)
└── data/                       ← VOLUME: persisted outside the container
    ├── calendars.json           ←   your calendar sources (edit this)
    ├── sync.log                 ←   sync history
    └── ics/
        ├── cal_1.ics            ←   synced calendar files
        └── cal_2.ics
```

Everything in `data/` is a Docker bind-mount. It persists across container rebuilds and image updates — your config and synced calendars are never overwritten.

## Configuring calendar sources

Edit `data/calendars.json`. It is created with an empty sources list on first run.

```json
{
  "sources": [
    {
      "id": 1,
      "name": "Family",
      "url": "https://calendar.google.com/calendar/ical/YOUR_ID/basic.ics",
      "filename": "cal_1.ics",
      "memberId": null
    },
    {
      "id": 2,
      "name": "Work",
      "url": "webcal://p62-caldav.icloud.com/published/2/...",
      "filename": "cal_2.ics",
      "memberId": null
    }
  ]
}
```

`webcal://` URLs are automatically converted to `https://` by sync.py.

After editing, trigger a sync:

```bash
docker compose exec home-calendar python3 /app/sync.py
```

### Getting your calendar URL

| Service | How to get the URL |
| --- | --- |
| Google Calendar | Settings → \[calendar name\] → Integrate calendar → *Secret address in iCal format* |
| Apple iCloud | Right-click calendar → Share Calendar → Public Calendar → copy the `webcal://` link |
| Nextcloud | Calendar → ··· → Copy private link |
| Outlook / Microsoft 365 | Calendar → Settings → Shared calendars → Publish → copy ICS link |

**Use the secret/private link, not the public one.** Public links require no auth token and may not include private events.

## Environment variables

Set in `docker-compose.yml` under `environment:`.

| Variable | Default | Description |
| --- | --- | --- |
| `TZ` | `Europe/London` | Container timezone — affects clock display and sync timestamps |
| `PORT` | `8080` | HTTP port inside the container |
| `SYNC_INTERVAL_MINUTES` | `15` | How often to re-fetch calendar feeds in the background |

To change the host port (e.g. to serve on port 80), edit the `ports:` mapping:

```yaml
ports:
  - "80:8080"
```

Then access the display at `http://<server-ip>/home-calendar.html`.

## Bin day setup

Look up your specific collection days at **leeds.gov.uk → bins → check your bin day** using your postcode. Leeds uses alternate-week schedules for green (recycling) and brown (garden) bins.

Open Settings (⚙) in the calendar and go to **Waste Collection — Garforth**.

| Bin | Colour | Typical schedule |
| --- | --- | --- |
| Black | General waste | Every week |
| Green | Recycling | Alternate weeks (A or B) |
| Brown | Garden waste | Alternate weeks (A or B) |

For each bin, set:

1. **Day of week** — when it is collected on your street.
2. **Frequency** — every week, or alternate week A / alternate week B.
3. **This week is** — whether the current calendar week is an A or B week (check against your last collection).

The display shows three visual states:

- **Inactive (greyed)** — not due this week
- **Week-active (coloured, gold border)** — due this week
- **Today/tomorrow (coloured, taller, bold gold border)** — due today or tomorrow

## Display features

### Top bar (40% of screen height)

Three equal panels across the full width:

- **System Clock** — UK time (24-hour by default, configurable). Updates every 10 seconds. Shows full date below.
- **Atmospheric** — Current conditions for Garforth, Leeds (53.7933°N, 1.3802°W) from the Met Office UKMO UKV 2km model via [Open-Meteo](https://open-meteo.com). Shows temperature, feels-like, wind speed, high/low, and a 5-hour forecast timeline. Refreshes every 30 minutes.
- **Waste Collection** — Bin icons for general (black), recycling (green), and garden (brown) bins with the three-state visual system described in the Bin day setup section.

### Calendar area (60% of screen height)

Three views, switchable from the nav bar:

- **Week** (default) — 7-day grid with hourly time slots. Current time shown as a gold line. Auto-scrolls to the current hour on load.
- **Month** — Full month grid with up to 3 events per day and overflow count.
- **Agenda** — Chronological list of upcoming events, configurable number of days ahead.

### Settings

Open via the ⚙ button in the calendar nav bar. Configure:

- Week start day and time format (12/24 hour)
- Family members with colour coding — each calendar source can be assigned to a member
- Calendar sources (iCal URLs or local file upload)
- Bin day schedule
- Agenda view range (days ahead)

Settings and synced event data persist in the browser's `localStorage`.

## Running without Docker

Requires Python 3.8 or later. No external packages — stdlib only.

```bash
# First-time setup
python3 setup.py

# Edit calendars.json with your calendar URLs
nano calendars.json

# Sync calendars
python3 sync.py

# Start the server (port 8080, syncs every 15 min in background)
python3 serve.py

# Custom port
python3 serve.py 9090
```

The server prints the local network URL on startup. Open that URL on the tablet.

## Scheduling and autostart

### Crontab (sync only)

If running without Docker, add a cron job to sync calendars periodically:

```bash
crontab -e
```

Every 15 minutes:

```
*/15 * * * * /usr/bin/python3 /home/pi/home-calendar/sync.py >> /home/pi/home-calendar/sync.log 2>&1
```

Find your Python path with `which python3`.

### systemd (server autostart on boot)

Create `/etc/systemd/system/home-calendar.service`:

```ini
[Unit]
Description=Home Calendar Server
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/home-calendar
ExecStart=/usr/bin/python3 /home/pi/home-calendar/serve.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable home-calendar
sudo systemctl start home-calendar
sudo systemctl status home-calendar
```

### Docker autostart

The `docker-compose.yml` already includes `restart: unless-stopped`. As long as Docker starts on boot (it does by default after installation), the container restarts automatically after reboots. No additional config needed.

## Tablet setup

The display is designed for a 10-inch Android tablet mounted permanently in a picture frame with the charging cable concealed in the frame.

### Recommended steps

1. **Enable developer options** — tap Build Number seven times in Settings → About phone.
2. **Stay awake while charging** — Settings → Developer options → Stay awake. This keeps the screen on while plugged in.
3. **Install a kiosk browser** — [Fully Kiosk Browser](https://www.fully-kiosk.com) is the most capable option. Set the start URL to `http://<server-ip>:8080/home-calendar.html`. Enable *Start on boot* and *Reload on idle*. Disable the address bar for a clean display.
4. **Reduce screen brightness** — set to a comfortable level for a dark room. The WY theme is designed for low ambient light.
5. **Disable notifications** — put the tablet in Do Not Disturb mode to prevent alerts appearing over the display.
6. **Set screen timeout to Never** — if not using a kiosk browser, ensure the screen does not time out.

### Frame mounting

Route the USB-C charging cable through the back of the picture frame before mounting. A right-angle USB-C adapter helps keep the cable flat against the wall.

## Troubleshooting

**Container won't start**

```bash
docker compose logs
```

**Sync failing for a specific calendar**

```bash
cat data/sync.log | grep -i error
```

Common causes: wrong URL (use the secret/private iCal link, not the public one), a `webcal://` URL that wasn't auto-converted, or the server temporarily unreachable.

**Events not updating on the tablet**

The HTML caches events in `localStorage`. After a sync completes, press the ↺ button in the calendar nav bar to reload from the local `.ics` files.

**Can't reach the calendar from the tablet**

- Confirm both devices are on the same Wi-Fi network.
- Check firewall: `sudo ufw allow 8080`
- Find the server IP: `hostname -I`

**Port 8080 already in use**

Change the host port in `docker-compose.yml`:

```yaml
ports:
  - "9090:8080"
```

Then access via `http://<server-ip>:9090/home-calendar.html`.

**Weather not loading**

The Open-Meteo API requires an internet connection. The location is fixed to Garforth, Leeds (53.7933°N, 1.3802°W) and uses the Met Office UKMO UKV 2km model. If unavailable, the weather panel shows "Unavailable" and retries on the next 30-minute refresh.

**Rebuilding after a file update**

```bash
docker compose up -d --build
```

Your `data/` folder (config and synced ICS files) is untouched.
