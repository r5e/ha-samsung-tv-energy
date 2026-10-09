# Changelog

## 1.0.2 (October 2026)

- **Fix:** after a Home Assistant restart, the window sensor could fail to load at all
  (`ValueError ... has the non-numeric value: 'None'` in the log), so the energy total stopped
  counting until the next restart. The 1.0.1 guard rendered the text "None", which a REST sensor
  with a unit rejects. The sensor now uses an `availability` template instead: it shows
  **unavailable** until the first successful read (normally within 5 minutes), then recovers on
  its own.
- **To update:** replace the window sensor's `value_template` block in your package with the
  `availability` and `value_template` lines from the current `samsung_tv_energy.yaml`, check the
  configuration, and restart.

## 1.0.1 (October 2026)

- The window sensor no longer logs a template error at startup, when its first read runs
  before Home Assistant's API is ready (`value_json is undefined`). It stays unknown until the
  next successful read.

## 1.0.0 (October 2026)

- First release: REST sensor reading the TV's 15-minute `deltaEnergy` windows from Home
  Assistant's SmartThings diagnostics, and a template sensor adding them up into an energy
  sensor for the Energy dashboard.
- Windows are counted by their `end` time, so Home Assistant restarts neither double-count
  nor skip a window.
