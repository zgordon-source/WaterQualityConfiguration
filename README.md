# Freeboard Sensor Utility: Yosemitech sensors

A single-file, browser-based replacement for the vendor configuration software for Yosemitech RS-485 Modbus water quality sensors. It talks Modbus RTU over an RS-485-to-USB adapter using the Web Serial API.

## Supported sensors
| Model | Measures | Serial prefix | Calibration |
|---|---|---|---|
| Y521-A | Conductivity, temperature, TDS, salinity | YL09 | 1-point (K) or 2-point (K, B) |
| Y504-A | DO saturation, DO mg/L, temperature | YL15 | 1-point at 100% (K) or 2-point 100%/0% (K, B) |
| Y511-A | Turbidity, temperature | YL29 | Zero (B), 1-point (K) or 2-point (K, B) |
| Y517-A | Oil in water, temperature | YL12 | Zero (B), 1-point (K) or 2-point (K, B) |
| Y532-A | pH, electrode mV, temperature | YL43 | Sensor-side 3-point (4.00 / 6.86 / 9.18) |

The sensor type is detected from the first four characters of the serial number after connecting, and can be set by hand. Y514-A (YL36), Y517-B (YL14) and Y516-C (YL63) are recognized by name but not supported yet; address changes still work for them.

## Requirements
- Google Chrome or Microsoft Edge on a desktop OS (Firefox and Safari lack Web Serial)
- Page served over `https://` (GitHub Pages) or `http://localhost`
- RS-485 adapter driver installed

## Deploy on GitHub Pages
1. Put `index.html` at the root of a repository.
2. Settings → Pages → Deploy from a branch, `main`, `/ (root)`.
3. Open `https://<org>.github.io/<repo>/`.

Local test: `python -m http.server 8000`, then open `http://localhost:8000`.

## Registers used (from the manuals)
Common: 0x3000 address, 0x0700 versions, 0x0900 serial number, 0x1100 user K/B.

| Model | Readings | Other |
|---|---|---|
| Y521 | 0x2600 (8): temp, conductivity mS/cm, TDS, salinity | 0x2000 TK/TB |
| Y504 | 0x2600 (6): temp, DO saturation, DO mg/L | 0x2000 TK/TB, 0x2700 cap parameters, 0x1500 salinity, 0x2400 air pressure |
| Y511 | 0x2600 (4): temp, NTU | 0x3100 run brush, 0x3200 brush interval (min), 0x1B00 brush on power-up |
| Y517 | 0x2600 (4): temp, OIW | same brush registers as Y511 |
| Y532 | 0x2600 (4): mV, pH; 0x2400 (2): temp | 0x2300 3-point calibration step, 0x0E00 status, 0x2900 K1–K6 |

Floats are little-endian (DCBA) per the manuals; a setting switches byte order if needed.

## Calibration math
K/B sensors are assumed to apply `reported = K × raw + B`. Each point averages the last 10 readings and recovers raw using the K and B in the sensor at the start:
- Zero-point: `B = S − K × raw`, K unchanged
- One-point: `K = (S − B) / raw`, B unchanged
- Two-point: `K = (S2 − S1) / (raw2 − raw1)`, `B = S1 − K × raw1`

This matches the Y504 manual's procedure (reset to K=1, B=0, then `K = 1/(Y − X)`, `B = −K·X`) without needing the reset.

pH: the sensor fits its own curve. At each buffer, once the mV reading is stable for a minute, the tool writes the buffer's pH to 0x2300, then reads the status from 0x0E00.

Validate each sensor type against the vendor software once before relying on it in the field.

## Calibration report
When a calibration finishes, the Verify & report step lets you record a verification reading in each standard, add an optional operator name and notes, and download:

- **Report (PDF):** sensor and session details, method and settling criterion, coefficients before and after, statistics for each calibration point (mean, SD, range, samples, temperature, time to capture, whether it was stable), verification results against the tolerance, settling charts, notes, and sign-off lines.
- **Raw data (JSON):** everything in the report plus every reading taken during each point.

The PDF footer prints the SHA-256 of the JSON file. Running `sha256sum` (or `Get-FileHash` on Windows) on the downloaded JSON confirms the data hasn't changed.

Verification tolerances default to each manual's stated accuracy and can be changed per calibration: Y521 ±1% or ±0.01 mS/cm, Y504 ±1% saturation, Y511 ±5% or ±0.3 NTU, Y532 ±0.02 pH, Y517 none.

The PDF is generated in the browser by jsPDF 2.5.2 (MIT license), embedded in `index.html`; no network access is used.

## Branding
Freeboard Technology logo and colors (navy #014F7F, sky #44AADB, yellow #FFC82D) are embedded in `index.html`. Colors are CSS variables at the top of the file.
