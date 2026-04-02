# NYC Urban Air Quality Monitor

A low-cost, open-source urban air quality monitoring system built on a Raspberry Pi. Captures particulate matter, temperature, humidity, and gas readings from outdoor sensors and streams them to a live browser-based dashboard — no cloud account, no backend server required.

---

## Why It Matters

Urban air quality is an invisible public health issue. PM2.5 — fine particulate matter smaller than 2.5 microns — penetrates deep into the lungs and is linked to respiratory disease, cardiovascular events, and premature death. Official EPA monitoring stations are sparse; a city block away from a monitor, readings can differ significantly due to traffic, construction, or localized pollution sources.

This project puts a calibrated sensor at street level, giving you hyperlocal data that government networks cannot provide. It also feeds into citizen science networks (Sensor.Community, OpenSenseMap), contributing to the global open air quality dataset.

---

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│  Raspberry Pi                                       │
│  Pimoroni Enviro+ HAT  +  PMS5003 sensor           │
│  sensor_reader.py  →  JSON stdout (every 30s)      │
└────────────────────┬────────────────────────────────┘
                     │  SSH tunnel (paramiko)
                     ▼
┌─────────────────────────────────────────────────────┐
│  Host machine (laptop / server)                     │
│  bridge.py streams and appends to daily CSV         │
│  readings_YYYY-MM-DD.csv                            │
│  Optional: push to Sensor.Community / OpenSenseMap  │
└────────────────────┬────────────────────────────────┘
                     │  CSV file
                     ▼
┌─────────────────────────────────────────────────────┐
│  Browser dashboard  (index.html)                    │
│  Chart.js  ·  EPA AQI  ·  auto-refresh every 30s   │
└─────────────────────────────────────────────────────┘
```

The entire pipeline runs without a database or cloud service. The dashboard reads the CSV directly from your local filesystem via the File System Access API and polls it on a configurable interval.

---

## Hardware

| Component | Purpose |
|-----------|---------|
| Raspberry Pi (3B+ or 4) | Edge compute, sensor host |
| [Pimoroni Enviro+ HAT](https://shop.pimoroni.com/products/enviro) | GPIO breakout for multiple sensors |
| BME280 | Temperature (°C), relative humidity (%), barometric pressure (hPa) |
| MICS6814 | Tri-gas sensor — oxidising, reducing, NH3 (raw resistance in kΩ) |
| LTR559 | Ambient light (lux) and proximity |
| [PMS5003](https://www.adafruit.com/product/3686) | Laser particulate counter — PM1.0, PM2.5, PM10 (µg/m³) |

---

## Features

### Dashboard
- **6 live metric cards** — PM1, PM2.5, PM10, temperature, humidity, pressure
- **EPA Air Quality Index (AQI)** — calculated from PM2.5 using official breakpoints; colour-coded needle gauge across six categories (Good → Hazardous)
- **Dynamic health insight** — contextual recommendation text that updates with AQI category
- **Three time-series charts** — PM2.5 trend, PM10 trend, dual-axis temperature & humidity
- **Time range controls** — 1 Hour / 24 Hours / All Data / Custom date-time range picker
- **Auto-refresh** — re-reads the CSV every 30 seconds; pause/resume without reloading the page
- **Temperature unit toggle** — switch between °C and °F
- **Dark mode** — full theme switch, preference saved to `localStorage`
- **Zero-install** — single `index.html` file; open it in a browser, load a CSV, done

### Data Acquisition
- SSH-based bridge — no special Pi configuration beyond key-based auth
- Daily-rotating CSV output (`readings_YYYY-MM-DD.csv`)
- One-command Pi setup script (`bridge/deploy.py`) — installs dependencies, enables I2C/SPI, pushes SSH key
- Synthetic data generator (`data/gen_sim.py`) — produces realistic diurnal patterns for testing without hardware
- Citizen science exporters for [Sensor.Community](https://sensor.community) and [OpenSenseMap](https://opensensemap.org)

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Sensor firmware | Python 3, `enviroplus`, `pms5003`, `bme280`, `ltr559`, `smbus2` |
| SSH bridge | Python 3, `paramiko` |
| Dashboard | HTML5, CSS3, vanilla JavaScript (no build step) |
| Charting | [Chart.js 4.4](https://www.chartjs.org) + `chartjs-adapter-date-fns` |
| Data format | CSV (comma-separated, ISO 8601 timestamps) |
| Deployment | Bash (`setup_pi.sh`), Python deploy script |

---

## Getting Started

### 1. Set up the Raspberry Pi

```bash
# From your host machine
cd air-monitor
python bridge/deploy.py --host <PI_IP> --user <PI_USER> --pass <PI_PASS>
```

This installs all sensor libraries, enables I2C/SPI, and configures passwordless SSH.

### 2. Start data collection

```bash
python bridge/bridge.py --host <PI_IP>
# Appends to ./data/readings_YYYY-MM-DD.csv every 30 seconds
```

### 3. Open the dashboard

Open `air-quality-monitor/index.html` in Chrome or Edge (File System Access API required for auto-refresh; Firefox/Safari fall back to manual file reload).

Click **Load CSV File** and select the CSV produced in step 2. The dashboard begins polling automatically.

### No hardware? Use synthetic data

```bash
python air-monitor/data/gen_sim.py
# Writes readings_YYYY-MM-DD.csv with realistic 2-day diurnal patterns
```

---

## CSV Format

```
timestamp,temperature,pressure,humidity,light,proximity,oxidising,reducing,nh3,pm1,pm2.5,pm10
2026-03-28T08:00:00Z,18.4,1013.2,61.5,4200,0,14.2,102.3,88.1,3.1,7.4,9.2
```

The dashboard auto-detects column names (case-insensitive, multiple aliases per field) and supports comma, semicolon, or tab delimiters.

---

## Limitations

- **No NO2 or VOC sensors.** The MICS6814 provides raw resistance values for oxidising and reducing gases but is not calibrated to return NO2 or CO concentrations in standard units. A dedicated NO2 sensor (e.g., MiCS-2714) or VOC sensor (e.g., SGP30) would be required for accurate gas monitoring.
- **Instantaneous AQI.** The EPA's official PM2.5 AQI is based on a 24-hour rolling average. This dashboard uses the most recent single reading as a proxy, which will overstate AQI during brief spikes and understate it during sustained pollution events.
- **No persistent storage.** Data lives in daily CSV files. There is no database, so historical queries spanning multiple days require manually loading multiple files.
- **Browser compatibility.** Auto-refresh via the File System Access API requires Chromium-based browsers. Firefox and Safari users must reload the file manually.
- **Local network only.** The SSH bridge assumes the Pi is on the same network (or reachable via VPN). Remote deployment requires additional tunnelling.
- **Sensor drift.** Low-cost sensors drift over time and are not traceable to reference standards. Results are indicative, not regulatory-grade.

---

## Future Improvements

- [ ] Add NO2 sensor (MiCS-2714 or Alphasense NO2-B43F)
- [ ] Add CO2 / VOC sensor (SCD40, SGP30, or BME688)
- [ ] Time-series database backend (InfluxDB or SQLite) to enable multi-day queries
- [ ] Grafana dashboard as a richer alternative to the HTML dashboard
- [ ] Alerting system — push notifications when AQI exceeds a threshold
- [ ] Calibration workflow against a co-located reference instrument
- [ ] Wind speed / direction integration (ultrasonic anemometer)
- [ ] GPS tagging for mobile deployments (bike-mounted sensor transects)
- [ ] Docker Compose stack to package bridge + database + dashboard together

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

## Acknowledgements

- [Pimoroni](https://pimoroni.com) for the Enviro+ HAT and Python libraries
- [Sensor.Community](https://sensor.community) for the citizen science data network
- [Chart.js](https://www.chartjs.org) for the charting library
- EPA for the public PM2.5 AQI breakpoint tables
