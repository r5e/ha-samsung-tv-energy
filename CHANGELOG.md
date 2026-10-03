# Changelog

## 1.0.0 (October 2026)

- First release: REST sensor reading the TV's 15-minute `deltaEnergy` windows from Home
  Assistant's SmartThings diagnostics, and a template sensor adding them up into an energy
  sensor for the Energy dashboard.
- Windows are counted by their `end` time, so Home Assistant restarts neither double-count
  nor skip a window.
