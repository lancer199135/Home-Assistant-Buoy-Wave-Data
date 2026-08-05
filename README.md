# NOAA/NDBC Buoy Dashboard for Home Assistant

Pull live wave, wind, and tide data straight from NOAA's National Data Buoy
Center (NDBC) into Home Assistant — no API key, no cloud service, just
free public text feeds NOAA already publishes.

This started as a personal setup for a specific buoy off the South
Carolina coast, but the sensors are written so the field logic works for
**any** NDBC buoy or C-MAN weather station — you just swap in your own
station ID(s).

## What you get

- **Wave conditions**: significant wave height, swell height/period/direction,
  wind-wave height/period/direction, wave steepness, and a live
  Rising/Falling/Steady trend indicator
- **Wind conditions**: speed, gust, and compass direction — for the buoy
  itself and optionally a second nearby wind-only station
- **Tides**: next/last tide time and type, current water level percentage
- **Temperature**: air and water temperature
- **A locally-cached, auto-refreshing spectral wave plot image** for your
  dashboard — refreshed on a schedule that matches NOAA's own update
  cadence, without hammering their servers or risking an IP ban (see
  [A note on the image](#a-note-on-the-image) below)

## Data source

Everything comes from NDBC's realtime text feeds:
`https://www.ndbc.noaa.gov/data/realtime2/<STATION_ID>.<TYPE>`

| Extension | Contents |
|---|---|
| `.txt` | Standard meteorological data (wind, pressure, air/water temp) |
| `.spec` | Spectral wave summary (height, period, direction, steepness) |
| `.swdir` | Raw spectral direction data (used here only for a reliable timestamp) |

Full field reference: https://www.ndbc.noaa.gov/faq/measdes.shtml

**Please poll responsibly.** NDBC's FAQ explicitly asks users to keep
retrievals minimal. The `scan_interval: 600` (10 minutes) on every REST
sensor here is deliberate — don't lower it. And never point a live/streaming
camera entity directly at an NDBC URL; Home Assistant polls those
continuously and it can look like abusive traffic (this happened during
development of this config and resulted in a temporary IP ban).

## Installation

1. **Find your station ID(s)** at https://www.ndbc.noaa.gov/ — search by
   name or click the map. Confirm your station actually reports what you
   want (not every buoy has wave sensors) by opening
   `https://www.ndbc.noaa.gov/data/realtime2/YOURID.txt` in a browser first.
2. **Copy `sensors.yaml`'s contents** into your `configuration.yaml`, or
   reference it via a `!include`.
3. **Find and replace** every occurrence of `41004` (primary buoy),
   `41076` (secondary buoy), and `FBIS1` (secondary wind station) with your
   own station ID(s). Delete any sensor blocks you don't need.
4. **Update the tide sensors'** `station_id` values to your local NOAA
   CO-OPS tide station — find one at https://tidesandcurrents.noaa.gov/
   (this is a different ID system than the buoy IDs above).
5. Restart Home Assistant.

### Optional: the auto-refreshing spectral wave image

NOAA also publishes a rendered wave-spectrum plot as an image, which is
handy to show on a dashboard — but it requires a different approach than
a simple sensor, for a reason worth understanding before you set it up.

**Don't use a live camera entity pointed at NOAA's URL.** It's tempting,
since Home Assistant has a built-in `generic` camera platform that just
takes a `still_image_url`. The problem is that HA polls camera entities
continuously — every few seconds, per dashboard viewer — with no
built-in throttling. That's the opposite of the "poll responsibly"
guidance above, and it's exactly what caused a temporary IP ban from
NOAA during development of this project. The fix is to download the
image to your *own* server on a schedule, and only ever display that
local copy.

**1. Set up the Downloader integration.** In current Home Assistant
versions this is UI-only (no YAML config): go to **Settings → Devices &
Services → Add Integration → Downloader**, and set the download
directory to `www` (this maps to `/config/www/`, which HA automatically
serves at `/local/...`).

**2. Add an automation to fetch the image on NOAA's actual update
schedule.** Check your buoy's plot page in a browser over time to find
its real cadence — this example buoy updates at `:20` and `:50` past the
hour, so the automation below fires 2 minutes later to be safe:

```yaml
alias: "Fetch 41004 Spectral Image"
trigger:
  - platform: time_pattern
    minutes: "22"
  - platform: time_pattern
    minutes: "52"
action:
  - service: downloader.download_file
    data:
      url: "https://www.ndbc.noaa.gov/plot?station=41004&meas=spec"
      filename: "41004_spec.png"
      overwrite: true
```

Add this through **Settings → Automations → Add Automation → Edit in
YAML**, pasting only the content above (no `automation:` key, no leading
`- `, since that editor already supplies both).

**3. Display it with a Markdown card, not a `picture` card.** This part
matters: plain `picture` cards in Lovelace don't reliably re-evaluate
Jinja templates in their `image:` field, so a cache-busting query string
never actually changes and your browser just keeps serving the same
cached image forever, even though the file on disk is updating fine.
A Markdown card renders templates properly:

```yaml
type: markdown
content: >-
  <a href="https://www.ndbc.noaa.gov/show_plot.php?station=41004&meas=spec&uom=E" target="_blank">
  <img src="/local/41004_spec.png?v={{ now().timestamp() | int }}" style="width:100%;">
  </a>
```

The `?v={{ now().timestamp() | int }}` forces a fresh fetch of your
*local* file on every card render — since it's just a local file read,
this has zero cost to NOAA no matter how often the card refreshes.

## Why some templates hold their last value

NOAA buoys frequently report `MM` (missing measurement) for individual
fields — especially swell period/direction, which need enough energy
separation from wind-waves to calculate. Rather than flashing to
`unknown` every time one field is briefly missing, several sensors here
hold their last known good value until fresh real data arrives, using
Home Assistant's `this.state` template variable. This is intentional
behavior, not a workaround for a bug.

## A note on the wave trend sensor

`41004 Wave Height Trend` is written as a **trigger-based** template
sensor rather than a regular reactive one — it only recalculates when the
wave height value actually *changes*, using `trigger.from_state` /
`trigger.to_state` for a true previous-vs-current comparison. This keeps
it accurate regardless of how often the underlying data happens to poll.

## Adapting field indices for a different station

Field positions (`data[N]` in the templates) come from NDBC's fixed
column order for each file type, which is the same across all stations
using that file type — so once a sensor works for one buoy, the same
index works for any other buoy of the same file type without changes.
Only the station ID needs to change.

## License

This config is shared as-is, free to copy, modify, and redistribute.
Feel free to open an issue or pull request if you adapt it for another
buoy and want to share improvements back.

## Credit

All data courtesy of NOAA's National Data Buoy Center
(https://www.ndbc.noaa.gov). This project is not affiliated with or
endorsed by NOAA.
