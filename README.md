# 🌞 Live Space Weather Dashboard — Power BI

A Power BI dashboard that monitors **space weather in near real time** using public data from NOAA's [Space Weather Prediction Center (SWPC)](https://www.swpc.noaa.gov/). It tracks geomagnetic activity, the solar wind, solar flares and official NOAA alerts — the same conditions that can disrupt GPS, radio communications, satellites and power grids.

---

## Why this project

Most portfolio dashboards are built on static CSV files. This one connects directly to a **live public JSON API**, cleans and models the data in Power Query, and refreshes automatically — the same workflow used for operational reporting in industry.

Space weather is a good test case: the data arrives from several feeds, at different update rates (1-minute to 3-hour), in more than one JSON layout, and the source changed during development.

---

## 📊 What the dashboard shows

| Visual | What it tells you |
|---|---|
| **Current Kp** | Planetary K-index (0–9), the main measure of geomagnetic disturbance |
| **G Now** | NOAA geomagnetic storm level (G0 = none, G1 minor … G5 extreme) |
| **Current Bz (nT)** | North–south direction of the solar wind's magnetic field. Sustained **negative (southward) Bz** is what drives geomagnetic storms |
| **Solar Wind Speed (km/s)** | Typical is ~300–500 km/s; 600+ km/s indicates a fast stream or CME |
| **Current X-ray Class** | Solar flare class from GOES satellites (A, B, C, M, X — each step is 10× stronger) |
| **Kp bar chart** | 7-day Kp history, color-coded: 🟩 quiet · 🟨 active (Kp ≥ 4) · 🟧 storm (Kp ≥ 5) · 🟥 severe (Kp ≥ 7) |
| **Bz line chart** | 24-hour IMF Bz with a zero reference line |
| **X-ray flux chart** | 24-hour GOES 0.1–0.8 nm flux on a **log scale**, so flares stand out |
| **Solar wind speed chart** | 24-hour solar wind speed at 1-minute resolution |
| **Data As Of** | Timestamp of the latest data in each feed and of the last refresh, so viewers know how fresh it is |

---

## 🛰️ Data sources

All data is public, free and requires no API key.

| Table | NOAA SWPC endpoint | Update rate |
|---|---|---|
| `Kp` | `products/noaa-planetary-k-index.json` | Every 3 hours |
| `SolarWindPlasma` | `json/rtsw/rtsw_wind_1m.json` | ~1 minute |
| `SolarWindMag` | `json/rtsw/rtsw_mag_1m.json` | ~1 minute |
| `XRayFlux` | `json/goes/primary/xrays-1-day.json` | ~1 minute |
| `NoaaScales` | `products/noaa-scales.json` | As issued |
| `Alerts` | `products/alerts.json` | As issued |

Base URL: `https://services.swpc.noaa.gov/`
All timestamps are in **UTC**.

---

## 🛠️ How it's built

```
NOAA SWPC JSON API  ──►  Power Query (M)  ──►  Data model  ──►  DAX measures  ──►  Report
                          • reusable fetch fn     • 7 tables     • KPI cards
                          • type conversion                      • color rules
                          • cleaning / filtering                 • freshness
```

### Power Query (M)

- `fnSwpcTable` — one reusable function that downloads any SWPC file and returns a table. It handles **both JSON layouts** NOAA uses (array-of-arrays with a header row, and array-of-objects), so every feed is loaded the same way.
- Uses a **fixed base URL with** `RelativePath` in `Web.Contents`, so scheduled refresh works in the Power BI Service without the "dynamic data source" error.
- `fnNum` **/** `fnUtc` helpers convert text to numbers and timestamps safely using a fixed culture (`en-US`), so the report works regardless of the user's regional settings.
- Derived columns: storm level (`g_scale`) from Kp, flare letter and class label (e.g. `M2.3`) from X-ray flux, and a headline and type extracted from each alert's message text.

### DAX

- **Latest-value KPIs** with `LASTNONBLANKVALUE`, so cards always show the most recent reading rather than an average.
- **Flare classification** from raw flux (`Peak X-ray Class (24h)`).
- **Conditional-formatting measures** that return hex colors (`Kp Bar Color`, `Bz Color`, `Speed Color`, `X-ray Color`).
- **Fallback logic** in `G Now`: uses NOAA's official scale and falls back to the Kp-derived level if the scales feed has no current value.
- **Time measures**: last refresh time, and current time in UTC and US Central with automatic daylight-saving handling.

---

## 🧩 Challenges and how I solved them

**1. A data source was retired mid-project.**
The original solar wind files (`products/solar-wind/plasma-1-day.json` and `mag-1-day.json`) started returning **404 Not Found**. SWPC had moved to a new solar wind source in 2026. I switched to the new real-time endpoints (`json/rtsw/rtsw_wind_1m.json` and `rtsw_mag_1m.json`) and mapped the new field names (`proton_speed`, `proton_density`, `proton_temperature`) back to the existing column names, so **no DAX measures or visuals had to change**.

**2. The new feeds mix several spacecraft.**
The replacement files contain readings from more than one spacecraft at the same timestamps. I filtered to the spacecraft SWPC flags as `active`, which avoids duplicate points and zig-zagging lines.

**3. Date hierarchies broke the charts.**
Power BI's automatic date/time feature collapsed minute-level data into a single "Year" bar and summed the values. I turned off **Auto date/time** and used averages instead of sums, so charts plot the true time series.

**4. Very different value ranges.**
X-ray flux spans several orders of magnitude (10⁻⁸ to 10⁻³ W/m²). A **log-scale Y axis** makes both quiet periods and flares readable.

---

## 🔄 Refresh and "real time"

The report uses **Import mode**: each refresh downloads a fresh snapshot from NOAA.

| Option | Freshness |
|---|---|
| Power BI Desktop → **Refresh** | On demand |
| Power BI Service, **Pro** scheduled refresh | Up to 8× per day |
| Power BI Service, **Premium / PPU** scheduled refresh | Up to 48× per day (every 30 min) |

No gateway is needed because the source is a public web API with anonymous access. True minute-by-minute streaming would need a streaming setup such as Microsoft Fabric Real-Time Intelligence (see [Future improvements](#-future-improvements)).

---

## 📁 Repository structure

```
├── Space Weather Dashboard.pbix      # Power BI report
├── powerquery/
│   └── space_weather_queries.pq      # All Power Query (M) queries and helper functions
├── dax/
│   └── space_weather_measures.dax    # All DAX measures
├── images/
│   └── dashboard.png                 # Screenshot used in this README
└── README.md
```

---

## 🔮 Future improvements

- Stream solar wind data with **Microsoft Fabric Real-Time Intelligence** for true minute-level updates.
- Add a **27-day outlook** and **aurora forecast** (SWPC OVATION model).
- Store history in a database to analyze storms over months, not just the last 1–7 days.
- **Alert notifications** with Power Automate when Kp ≥ 5 or an M/X-class flare occurs.

---

## 📚 Glossary

- **Kp index** — 0–9 scale of global geomagnetic disturbance, measured every 3 hours.
- **IMF Bz** — North–south component of the interplanetary magnetic field carried by the solar wind. Negative = southward = energy flows into Earth's magnetosphere.
- **X-ray flare classes** — A < B < C < M < X, each 10× the previous. M and X flares can cause radio blackouts.
- **NOAA scales** — R (radio blackouts), S (solar radiation storms), G (geomagnetic storms), each from 1 (minor) to 5 (extreme).

---

## 🙏 Acknowledgements

Data: **NOAA Space Weather Prediction Center** (public domain, U.S. Government work). This project is independent and not affiliated with or endorsed by NOAA.
