# Samsung TV energy in Home Assistant (SmartThings workaround)

Get a Samsung TV's **real energy use** into Home Assistant's Energy dashboard, when the
SmartThings integration shows the TV's energy sensors as **unavailable**, even though the
Samsung Energy (SmartThings) app shows the TV's daily usage.

**Validated:** over five days, the daily totals in Home Assistant matched the Samsung
Energy app to within about 1 %.

**Tested on:** Samsung UA75BU8000 (2022 Crystal UHD BU8000), Home Assistant 2026.9. If it
works, or doesn't, on your TV, please [open an issue](https://github.com/r5e/ha-samsung-tv-energy/issues) with the
model, so others know what to expect.

> This is a workaround that reads Home Assistant's own diagnostics data. It isn't an
> official API, so a future Home Assistant or SmartThings change could break it. It
> changes nothing in SmartThings or on the TV; it only reads. Once the SmartThings
> integration handles these TVs properly (see
> [#150059](https://github.com/home-assistant/core/issues/150059)), this workaround won't
> be needed.

---

## Contents

1. [The problem](#1-the-problem)
2. [What's actually going on](#2-whats-actually-going-on)
3. [How the workaround works](#3-how-the-workaround-works)
4. [Does my TV qualify?](#4-does-my-tv-qualify)
5. [Setup](#5-setup)
6. [Checking it works](#6-checking-it-works)
7. [Validation results](#7-validation-results)
8. [Limitations](#8-limitations)
9. [Troubleshooting](#9-troubleshooting)
10. [What I found about standby power](#10-what-i-found-about-standby-power)

---

## 1. The problem

The SmartThings integration creates energy and power sensors for supported Samsung
devices. For some TVs these sensors stay **unavailable** (or read 0), even though the
Samsung Energy app shows the TV's usage day by day. This is reported in Home Assistant
issue [#150059](https://github.com/home-assistant/core/issues/150059) (a QN900C), and I
hit the same thing with a 2022 Crystal UHD **BU8000** (UA75BU8000).

## 2. What's actually going on

Downloading the TV's diagnostics from Home Assistant (Settings > Devices & services >
SmartThings > the TV > **Download diagnostics**) shows two things:

1. **The TV does report real energy,** in the `powerConsumptionReport` capability. Each
   report covers a **15-minute window** (`start` and `end` times), and the useful figure
   is **`deltaEnergy`**: the energy used in that window, in Wh. The cumulative `energy`
   and the `power` fields are always 0 on my TV, which is why a sensor built on them
   shows nothing useful.
2. **The TV lists `powerConsumptionReport` in `custom.disabledCapabilities`,** while still
   sending real data. The SmartThings integration respects that list, so it doesn't
   create working sensors for the capability. That's the root cause of the
   "unavailable" sensors.

So the data is there; it just never reaches a sensor.

## 3. How the workaround works

Two pieces, in one package file:

1. **A REST sensor** reads the TV's diagnostics from Home Assistant's own API every 5
   minutes, and exposes the latest window: `deltaEnergy` as its state (Wh), with the
   window's `start` and `end` times as attributes.
2. **A trigger-based template sensor** adds up the windows into a normal **energy
   sensor** (kWh, `total_increasing`) that the Energy dashboard accepts. Each window is
   counted **exactly once**, by remembering the `end` time of the last window counted.
   That makes it safe across restarts: a window that was already counted isn't counted
   again, and a new window that arrived while Home Assistant was restarting isn't lost.

## 4. Does my TV qualify?

Download the TV's diagnostics (as in section 2), open the file, and search for
`deltaEnergy`.

- **Present, with values that change over time:** this workaround should work.
- **Missing:** the TV doesn't report energy at all. My 2019 RU7100 is like this, and
  there's nothing to read.

Also check that the Samsung Energy app shows usage for the TV. If it does, the data
exists.

Other Samsung appliances (for example a washing machine) may report energy correctly
through the normal integration, in which case you don't need this.

## 5. Setup

### 5.1 Create a long-lived access token

In Home Assistant, open your **Profile** (bottom left), go to the **Security** tab, and
under **Long-lived access tokens** click **Create token**. Name it, for example, "TV
diagnostics poller". Copy it straight away (it's only shown once), and store it in your
password manager.

Add it to `secrets.yaml`, with the word `Bearer` and a space in front:

```yaml
ha_diag_token: "Bearer eyJhbGciOi..."
```

### 5.2 Find the two IDs

The REST sensor's address needs the SmartThings **config entry ID** and the TV's
**device ID**:

- **Config entry ID:** Settings > Devices & services > **SmartThings**. The address bar
  then ends in `...#config_entry=<ID>`, or open the entry's three-dot menu and check its
  details. It's a 26-character code of letters and digits.
- **Device ID:** open the TV's device page. The address ends in `/config/devices/device/<ID>`,
  a 32-character hex code.

**A quick check:** this address, opened in a browser where you're logged in to Home
Assistant, should download the same diagnostics as the button does:
`https://<your-ha>/api/diagnostics/config_entry/<CONFIG_ENTRY_ID>/device/<DEVICE_ID>`

### 5.3 Add the package

1. Make sure packages are enabled in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Copy [`samsung_tv_energy.yaml`](samsung_tv_energy.yaml) to `/config/packages/`, and
   edit the lines marked `EDIT`:
   - **the resource address:** your Home Assistant address, plus the two IDs;
   - **the sensor names** (and the matching `entity_id` in the trigger).
3. Developer Tools > YAML > **Check configuration**, then restart.

**Which address to use:** use the address you normally reach Home Assistant on. If Home
Assistant itself serves HTTPS (with a certificate for a domain name), `https://127.0.0.1`
will fail the certificate check. Either use the domain name, or keep `127.0.0.1` and add
`verify_ssl: false` to the REST entry. If you use plain HTTP on port 8123,
`http://127.0.0.1:8123` works and doesn't depend on DNS.

### 5.4 Add it to the Energy dashboard

Once the total is counting (section 6), add the **energy** sensor (not the window
sensor) under Settings > Dashboards > Energy > **Individual devices**.

## 6. Checking it works

- **The window sensor** shows a number in Wh, with `start` and `end` attributes. Right
  after a restart it can show **unknown** for up to 5 minutes (the REST sensor's first
  read can run before Home Assistant's API is ready).
- **The energy sensor** steps up whenever a new window arrives, normally every 15
  minutes. With the TV off, expect small steps (about 0.004 kWh each on my TV); with it
  on, larger ones (about 0.015–0.02 kWh).
- **After a day or two,** compare the daily totals with the Samsung Energy app. A
  Statistics graph card (period **Day**, stat type **Change**) shows the daily figures.

## 7. Validation results

Five days on a UA75BU8000, Home Assistant against the Samsung Energy app:

| Day | Samsung Energy app | Home Assistant | Difference |
|---|---|---|---|
| 27 Sep | 411 Wh | 407 Wh | −1.0 % |
| 28 Sep | 408 Wh | 404 Wh | −1.0 % |
| 29 Sep | 408 Wh | 404 Wh | −1.0 % |
| 30 Sep | 549 Wh | 545 Wh | −0.7 % |
| 1 Oct | 452 Wh | 453 Wh | +0.2 % |

These results come from the first version of the template. The steady −4 Wh is one
standby window a day. The most likely cause is that the first version skipped the first
new window after a Home Assistant restart (I restarted often during those days), while
the 1 October figure, a day without restarts, matches. The current template counts
windows by their end time instead, which avoids that, as described in section 3.

## 8. Limitations

- **Unofficial data source.** It reads Home Assistant's diagnostics, which isn't a stable
  API. A future change to the diagnostics format, or to the SmartThings integration,
  could break it.
- **15-minute resolution.** The TV reports energy in 15-minute windows, so this gives
  energy, not live power.
- **Missed windows.** If Home Assistant is down for longer than a window (15 minutes),
  the windows reported in the meantime aren't counted. Short restarts are fine.
- **The TV must stay connected to SmartThings.** If the TV drops off SmartThings, nothing
  is reported (by Samsung's app either). See below.
- **A token with full access.** Long-lived tokens have the same rights as your user. Keep
  it in `secrets.yaml`, and revoke it if you remove the package.

## 9. Troubleshooting

**The window sensor is unavailable or unknown for more than 10 minutes.**
Check Settings > System > Logs for errors from `rest`. The usual causes are the address
(port, HTTP versus HTTPS), the certificate (see 5.3), or the token line (it needs
`Bearer ` in front).

**The energy sensor stays at 0.**
It only counts **new** windows, so it starts with the next window after setup. If it's
still 0 after 30 minutes, check that the trigger's `entity_id` matches the window
sensor's actual entity ID.

**The window sensor keeps showing the same value for hours.**
The TV has probably stopped reporting to SmartThings: check the TV in the SmartThings
app. If it shows **offline** even though the TV's own network test is fine, **restart
the TV fully from the remote: hold the power button for about 5–10 seconds** until it
switches off and back on. That fixed it for me without unplugging a wall-mounted TV. If
it still shows offline, remove the TV in the SmartThings app and add it again, then check
whether its device ID in Home Assistant has changed (section 5.2).

**A reading of 0 appears now and then between normal readings.**
I saw occasional empty windows, for example around the TV switching on or off. They
don't affect the totals noticeably.

## 10. What I found about standby power

With the TV **off**, it reported a fixed **4.297 Wh every 15 minutes: about 17 W**, or
**0.40 kWh a day**, all day, every day. Watching TV adds only about 50–60 W on top. So on
an average day, the TV used more energy switched off than while being watched: around
**147 kWh a year** in standby alone.

The likely cause is a feature that keeps the TV half awake for a quick start and remote
wake-up: **Instant On** (Settings > General > Power and Energy Saving) on 2022 models,
and possibly the network standby options (**Power On with Mobile** or **IP Remote**).
Turning them off typically means a slower start, and that SmartThings or Home Assistant
can't wake the TV. Measuring standby before and after the change, with this workaround,
shows whether it's worth it for your TV.

---

## Licence

MIT. See [LICENSE](LICENSE). Not affiliated with Samsung or Home Assistant.
