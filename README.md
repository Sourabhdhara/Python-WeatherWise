# WeatherWise — Full Documentation

**A colorful, pure-Python desktop weather application built with Tkinter.**

---

## Table of Contents

1. [Overview](#1-overview)
2. [Features](#2-features)
3. [Tech Stack](#3-tech-stack)
4. [Project Structure](#4-project-structure)
5. [Installation & Setup](#5-installation--setup)
6. [How to Use the App](#6-how-to-use-the-app)
7. [Architecture & Data Flow](#7-architecture--data-flow)
8. [Code Walkthrough (Function by Function)](#8-code-walkthrough-function-by-function)
9. [UI Design System](#9-ui-design-system)
10. [Threading Model (Why the App Never Freezes)](#10-threading-model-why-the-app-never-freezes)
11. [External APIs Used](#11-external-apis-used)
12. [Error Handling](#12-error-handling)
13. [Customization Guide](#13-customization-guide)
14. [Known Limitations](#14-known-limitations)
15. [Possible Future Enhancements](#15-possible-future-enhancements)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. Overview

WeatherWise is a **desktop GUI weather application** written entirely in Python. It has no
web server, no browser, and no HTML/JavaScript — the entire interface is drawn using
**Tkinter**, the GUI toolkit built into every standard Python installation.

The app can:
- Automatically detect the user's approximate location via their IP address
- Look up weather for any city typed in by name
- Display current conditions, a 12-hour forecast, and a 7-day forecast
- Convert between Celsius/Fahrenheit and km/h/mph on the fly

Visually, it follows a bright, playful design system (light blue background, blue title
banner, colorful raised buttons, pale-yellow "card" panels) — the same visual language as
a simple To-Do List Tkinter app, applied consistently across a more feature-rich screen.

---

## 2. Features

| Feature | Description |
|---|---|
| 📍 Auto-detect location | On launch, the app calls an IP-geolocation service to guess the user's city and immediately loads its weather. |
| 🔍 City search | Type any city name and press Enter or click Search to load its weather. |
| 🌡️ Current conditions tab | Shows a large weather icon, current temperature, condition text, wind speed, humidity, and "feels like" temperature. |
| ⏱️ Hourly forecast tab | A horizontally scrollable strip of cards showing the next 12 hours (time, icon, temperature, wind). |
| 📅 7-day forecast tab | A row of cards, one per day, showing day name, date, icon, high/low temperature, and max wind speed. |
| 🔄 Refresh | Re-fetches weather for whatever location is currently loaded. |
| °C / °F toggle | Instantly redraws all displayed temperatures in the chosen unit — no new network call needed. |
| km/h / mph toggle | Same idea, for wind speed. |
| Non-blocking network calls | All internet requests run on background threads, so the window stays responsive at all times. |
| Friendly error handling | Pop-up messages for "city not found," network failures, and failed auto-detection. |

---

## 3. Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3 |
| GUI toolkit | `tkinter` + `tkinter.ttk` (both built into Python — no install needed) |
| HTTP client | `requests` (the only third-party dependency) |
| Concurrency | `threading` (built into Python) |
| Date/time handling | `datetime` (built into Python) |
| Weather data | [Open-Meteo](https://open-meteo.com) forecast API (free, no API key) |
| City → coordinates | Open-Meteo Geocoding API (free, no API key) |
| IP → location | [ip-api.com](https://ip-api.com) (free, no API key) |

No Flask, no web server, no database, and no API keys are required anywhere in this project.

---

## 4. Project Structure

```
WeatherWise.py   ← the entire application (single file)
```

The whole app intentionally lives in **one file** so it's easy to read top-to-bottom and
easy to share. It is organized into four clearly separated sections:

```
1. Color palette & constants
2. Weather icon lookup function
3. Network functions (geolocation, geocoding, weather fetch)
4. WeatherApp class (everything related to the window and widgets)
```

---

## 5. Installation & Setup

### Requirements
- Python 3.8 or newer (Tkinter ships with the standard Python installer on Windows/macOS;
  on some Linux distros you may need `sudo apt install python3-tk`)
- An internet connection (the app calls live weather/geolocation APIs)

### Steps

```bash
# 1. Install the one dependency
pip install requests

# 2. Run the app
python weather_gui_colorful_full.py
```

A window titled **"WeatherWise"** will open automatically.

---

## 6. How to Use the App

1. **On launch**, the app tries to detect your location automatically and loads its
   weather right away — no action needed.
2. **To check a different city**, type its name into the "Enter City" box and either
   press **Enter** or click **🔍 Search**.
3. **To re-detect your current location**, click **📍 Auto Detect**.
4. **To reload the same location's data** (e.g. to get updated numbers), click
   **🔄 Refresh**.
5. **To change units**, click the °C/°F or km/h/mph radio buttons at any time — the
   currently displayed data updates instantly without a new network call.
6. Use the three tabs — **Current**, **Hourly**, **7-Day** — to switch between views.
7. Click **Exit ➜]** to close the app.

---

## 7. Architecture & Data Flow

```
                 ┌─────────────────────┐
                 │   User Interaction    │
                 │ (search / auto-detect │
                 │  / refresh / unit toggle) │
                 └──────────┬───────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │  Background Thread       │
              │  (geocode_city /         │
              │   get_location_from_ip)  │
              └──────────┬───────────────┘
                            │  lat, lon, place name
                            ▼
              ┌─────────────────────────┐
              │  Background Thread       │
              │  fetch_weather_data()    │
              │  → calls Open-Meteo API  │
              └──────────┬───────────────┘
                            │  raw JSON weather data
                            ▼
              ┌─────────────────────────┐
              │  self.root.after(0, ...) │
              │  hands data back to the  │
              │  MAIN thread safely      │
              └──────────┬───────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │  _update_current_tab()   │
              │  _update_hourly_tab()    │
              │  _update_daily_tab()     │
              │  → redraw widgets        │
              └─────────────────────────┘
```

**Key idea:** all slow work (network calls) happens off the main thread. All widget
updates happen back on the main thread. This is why the window never freezes, even on a
slow connection.

---

## 8. Code Walkthrough (Function by Function)

### `get_weather_icon(code)`
Takes Open-Meteo's numeric **WMO weather code** (e.g. `61` = rain, `0` = clear sky) and
returns a tuple of `(emoji, description)`. This is pure logic — no network, no GUI — so
it's the simplest function in the file and a good starting point for understanding the
data.

### `get_location_from_ip()`
Calls `ip-api.com` with no parameters; the service inspects the caller's IP address and
returns an approximate city. Returns `(lat, lon, "City, Country")`, or `(None, None, None)`
if it fails.

### `geocode_city(city_name)`
Calls Open-Meteo's Geocoding API with a city name (e.g. `"Jaipur"`) and returns the best
matching result's coordinates and display name. Returns `(None, None, None)` if no city
is found.

### `fetch_weather_data(lat, lon)`
The core data function. Sends one HTTP request to Open-Meteo's forecast endpoint asking
for:
- `current_weather` — right-now conditions
- `hourly` — temperature, weather code, wind speed, humidity for every hour, several days out
- `daily` — weather code and min/max temperature and max wind speed, one row per day, for 7 days

Returns the raw JSON dictionary exactly as Open-Meteo sends it; all further processing
happens in the `WeatherApp` class.

### `WeatherApp` class

| Method | Responsibility |
|---|---|
| `__init__` | Sets up window title/size/background, initializes state variables, calls the builder methods, and kicks off auto-detection. |
| `_setup_notebook_style()` | The only `ttk.Style` configuration in the app — recolors the tab bar to match the app's palette. |
| `_build_ui()` | Creates the title banner, entry frame, button frame, unit-toggle frame, status label, and the notebook (tab container). |
| `_build_current_tab()` / `_build_hourly_tab()` / `_build_daily_tab()` | Create the *empty* layout/containers for each tab (these run once, at startup). |
| `_show_loading()` | Resets all labels to placeholder text while new data is being fetched. |
| `_on_unit_change()` | Runs when a unit radio button is clicked; re-renders already-downloaded data in the new unit — no network call. |
| `_convert_temp()` / `_convert_wind()` | Pure math conversions between units. |
| `_update_current_tab()` / `_update_hourly_tab()` / `_update_daily_tab()` | Take the stored weather JSON and actually fill in / redraw the widgets (these can run many times, whenever new data arrives or units change). |
| `_fetch_and_display()` | Runs **inside a background thread** — calls `fetch_weather_data()`, stores the result, then schedules a UI update via `self.root.after(0, ...)`. |
| `auto_detect_location()` / `search_location()` / `refresh_weather()` | The three ways a weather lookup can be triggered by the user; each spins up its own background thread. |
| `_set_buttons_state()` | Disables the action buttons while a request is in flight, to prevent duplicate clicks. |

---

## 9. UI Design System

The app follows a small, consistent color palette (defined as constants at the top of the
file, so they're easy to change in one place):

| Constant | Hex/Name | Used for |
|---|---|---|
| `BG_MAIN` | `#d2dff7` (light blue) | Main window and frame backgrounds |
| `BG_TITLE` | `#5c8fed` (medium blue) | Title banner, selected tab, location text |
| `BG_CARD` | `#fdf2b3` (pale yellow) | Weather cards (current/hourly/daily) — mirrors a listbox's "content area" look |
| `BTN_GREEN` | `"green"` | Search button |
| `BTN_ORANGE` | `#ff8c42` | Auto Detect button |
| `BTN_BLUE` | `#1e90ff` | Refresh button |
| `BTN_GRAY` | `#717274` | Exit button |
| `FONT_FAMILY` | `"Comic Sans MS"` | Every piece of text in the app |

All widgets are **plain `tk` widgets** (not `ttk`) so their colors can be set directly via
`bg=`/`fg=`, except for the **tab bar**, which requires `ttk.Notebook` (Tkinter has no
plain alternative for tabs) — that one widget's colors are set through `ttk.Style` instead.

---

## 10. Threading Model (Why the App Never Freezes)

Tkinter runs on a single main thread. If a slow operation (like a network request) runs
directly inside a button's `command=`, the entire window stops responding until it
finishes — no dragging, no clicking, nothing redraws.

WeatherWise avoids this with a consistent three-step pattern used by every action
(search, auto-detect, refresh):

```python
def search_location(self):
    # 1. Runs on the MAIN thread: read input, update status text
    city = self.city_entry.get().strip()
    ...
    # 2. Start a BACKGROUND thread for the slow network work
    threading.Thread(target=geocode_and_fetch, daemon=True).start()

def geocode_and_fetch():
    # Runs on the BACKGROUND thread — safe to block here
    lat, lon, name = geocode_city(city)
    # 3. Hand control back to the MAIN thread to touch widgets
    self.root.after(0, lambda: self._start_weather_fetch(lat, lon, name))
```

`self.root.after(0, callback)` is the one Tkinter-safe way to say *"run this function on
the main thread as soon as it's free."* Every place in the code that updates a label,
button, or card goes through this handoff.

---

## 11. External APIs Used

| API | Purpose | Auth required? | Docs |
|---|---|---|---|
| Open-Meteo Forecast API | Current + hourly + daily weather | No | https://open-meteo.com/en/docs |
| Open-Meteo Geocoding API | City name → coordinates | No | https://open-meteo.com/en/docs/geocoding-api |
| ip-api.com | IP address → approximate location | No (free tier, rate-limited) | https://ip-api.com/docs |

All three are free and require no signup or API key, which is why the app has no
configuration file or credentials to manage.

---

## 12. Error Handling

| Situation | What happens |
|---|---|
| City not found | A warning popup: *"📭 Could not find city: '...'"* |
| No internet / request timeout | A warning popup and the status bar shows *"❌ Error: ..."* |
| IP auto-detection fails | A popup suggesting the user search manually |
| Refresh clicked with no location loaded yet | An info popup: *"No location yet. Use 'Auto Detect' or 'Search' first."* |
| Empty search box | A warning popup asking for a city name |

All network calls are wrapped in `try/except`, so a failure never crashes the app — it
always degrades to a message box and a reset "loading" state.

---

## 13. Customization Guide

Because colors and fonts are defined once as constants, common tweaks are quick:

- **Change the color scheme:** edit the `BG_MAIN`, `BG_TITLE`, `BG_CARD`, `BTN_*`
  constants near the top of the file.
- **Change the font:** edit `FONT_FAMILY` (make sure the font is installed on the target
  machine, or it will silently fall back to a system default).
- **Change window size:** edit `self.root.geometry("900x650")` in `__init__`.
- **Show more/fewer hourly cards:** change the `12` in
  `end_idx = min(start_idx + 12, len(times))` inside `_update_hourly_tab()`.
- **Add a window icon:** add a line like
  `self.root.iconphoto(True, tk.PhotoImage(file="logo.png"))` right after
  `self.root.config(bg=BG_MAIN)` in `__init__`.

---

## 14. Known Limitations

- Weather and location data depend entirely on free third-party APIs; if any of them are
  down or rate-limit the request, that feature will show an error until it recovers.
- IP-based location is only approximate (city-level, sometimes off by a wider region) —
  it is not GPS-accurate.
- The app has no persistence: closing and reopening it forgets the last searched city and
  unit preferences, and starts over with auto-detection.
- `Comic Sans MS` may not be installed on all operating systems (it ships by default on
  Windows; on Linux/macOS it may fall back to a default font).

---

## 15. Possible Future Enhancements

- Remember the last-searched city and chosen units between sessions (e.g. saved to a
  small local JSON file)
- A "favorite cities" list, similar in spirit to a To-Do list's item list
- A window/taskbar icon
- Dark mode toggle alongside the existing colorful theme
- Weather alerts / severe weather warnings
- Packaging into a standalone `.exe`/`.app` with PyInstaller so it can run without a
  Python installation

---

## 16. Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'requests'` | Dependency not installed | Run `pip install requests` |
| Window opens but nothing loads | No internet connection, or a firewall blocking outbound requests | Check your network connection; check firewall settings |
| Tkinter fails to import on Linux | Tkinter isn't bundled with your Python build | Run `sudo apt install python3-tk` (Debian/Ubuntu) or the equivalent for your distro |
| Fonts look different than expected | `Comic Sans MS` isn't installed on your OS | Install the font, or change `FONT_FAMILY` to something available (e.g. `"Arial"`) |
| "City not found" for a real city | Spelling mismatch, or a very small/ambiguous place name | Try a larger nearby city, or add the country (e.g. `"Springfield, USA"` won't work — Open-Meteo's geocoder takes plain names only, so try just `"Springfield"`) |
