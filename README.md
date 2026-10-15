# NILAI · நிலை (steadiness)

**Non-invasive AI wearable that tells hand tremor apart from intentional movement, in real time, from forearm EMG + motion signals.**

Team **Nilai** · Hack for Social Cause 2026 · Team ID `HSC|TN|00325` · Sri Sairam Engineering College, Chennai, Tamil Nadu
Theme: Assistive Technology & Accessibility · UN SDG 3, 9, 10

![Live dashboard](docs/dashboard.png)

---

## What it does

| Step | What happens | Status |
|---|---|---|
| **Sense** | Forearm EMG + 3-axis gyroscope (MPU6050) sampled at 250 Hz by an ESP32 | ✅ working |
| **Distinguish** | Window-by-window classifier labels *rest / intentional movement / tremor / external vibration* | ✅ working |
| **Steady twin** | WFLC adaptive filter estimates the tremor and shows the movement with tremor removed | ✅ working |
| **Score** | 0–100 tremor score, tremor frequency, EMG fatigue (median frequency) on a live dashboard | ✅ working |
| **Suppress** | Timed EMS counter-pulse, user-triggered | 🔜 roadmap — not in this repo |

The key idea: **real tremor is muscle-driven**. When the forearm shakes at 3.5–12 Hz *and* the EMG activation envelope oscillates coherently at the same frequency, it is tremor. When the gyro shakes but the muscles are not driving it (a bus, a power tool), coherence stays low and NILAI ignores it.

## Repository layout

```
nilai/                    Python package
  config.py               all settings, read from environment variables
  acquisition.py          data sources: synthetic | csv | serial (ESP32) | ble (NPG Lite)
  dsp.py                  filters, PCA gyro axis, Welch PSD, band power, coherence, EMG median freq
  classifier.py           feature extraction + tremor/voluntary/vibration/rest decision
  wflc.py                 Weighted-frequency Fourier Linear Combiner (tremor estimator)
  pipeline.py             sliding-window streaming + batch analysis
  synthetic.py            realistic synthetic EMG + gyro scenarios
  cli.py                  analyse / generate / dashboard commands
  dashboard/              Flask + Server-Sent Events live dashboard
firmware/esp32_imu_stream ESP32 / ESP32-C6 sketch: MPU6050 + analog EMG → USB CSV stream
data/sample/              sample recordings (rest, voluntary, tremor, vibration)
tests/                    pytest suite (25 tests)
docs/TECHNICAL_DESIGN.md  architecture, algorithms, diagrams
docs/pitch/               Hack for Social Cause presentation (PDF + PPTX)
tools/hsc-deck/           script that generates the presentation
```

## Quick start (no hardware needed)

Requires Python 3.10+.

```bash
git clone <this-repo-url> nilai && cd nilai
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

pytest -q                                               # 25 tests should pass
python -m nilai.cli analyse data/sample/tremor.csv      # classify a recording
python -m nilai.cli analyse data/sample/vibration.csv   # vibration is rejected
python -m nilai.cli dashboard                           # open http://127.0.0.1:5000
```

The dashboard defaults to the synthetic source and cycles through *rest → intentional movement → tremor → external vibration* so every state can be seen without a device.

Example output:

```json
{
  "file": "data/sample/tremor.csv",
  "windows": 35,
  "labels": { "tremor": 35 },
  "dominant": "tremor",
  "mean_tremor_score": 79.6,
  "mean_tremor_hz": 5.5
}
```

## Running with the wearable

1. Flash `firmware/esp32_imu_stream/esp32_imu_stream.ino` with Arduino IDE (board: *ESP32C6 Dev Module* or *ESP32 Dev Module*). Wiring is in the sketch header.
2. Keep the arm still for ~1 s after power-up (gyro bias calibration).
3. Run:

```bash
export NILAI_SOURCE=serial
export NILAI_SERIAL_PORT=/dev/ttyUSB0     # Windows: COM5, macOS: /dev/cu.usbserial-XXXX
python -m nilai.cli dashboard
```

To record a session for later analysis, save the serial stream (it is already CSV with a header) and run `python -m nilai.cli analyse your_file.csv` (convert the `t` column from ms to seconds first, or record via the `csv` source).

## Environment variables

Copy `.env.example` to `.env` (or export them in your shell). All are optional.

| Variable | Default | Meaning |
|---|---|---|
| `NILAI_SOURCE` | `synthetic` | `synthetic`, `csv`, `serial` or `ble` |
| `NILAI_SERIAL_PORT` | `/dev/ttyUSB0` | ESP32 serial port |
| `NILAI_SERIAL_BAUD` | `115200` | serial baud rate |
| `NILAI_CSV_PATH` | `data/sample/tremor.csv` | file replayed when source is `csv` |
| `NILAI_BLE_ADDRESS` | – | NPG Lite BLE MAC/UUID (source `ble`, needs `pip install bleak`) |
| `NILAI_BLE_CHAR_UUID` | – | NPG Lite notify characteristic |
| `NILAI_FS` | `250` | sampling rate (Hz) |
| `NILAI_WINDOW_S` | `3.0` | analysis window (s) |
| `NILAI_HOP_S` | `0.5` | window step (s) |
| `NILAI_REST_GYRO_RMS` | `2.0` | below this (deg/s) the arm is at rest |
| `NILAI_TREMOR_BAND_RATIO` | `0.5` | share of motion power in 3.5–12 Hz to consider tremor |
| `NILAI_COHERENCE_MIN` | `0.6` | EMG–gyro coherence needed to call it tremor |
| `NILAI_HOST` / `NILAI_PORT` | `127.0.0.1` / `5000` | dashboard address |

## Sample data and tests

`data/sample/` holds 20-second recordings for each scenario (columns `t, emg, gx, gy, gz`; t in s, EMG in ADC-centred units, gyro in deg/s). They are generated by `python -m nilai.cli generate` from `nilai/synthetic.py`, which models EMG as band-limited noise modulated by muscle activation and drives the gyro from that activation (tremor) or independently of it (vibration).

The test suite checks that:
- rest and intentional movement are never labelled tremor
- tremor at 4.5, 6, 8 and 10 Hz is detected (>95% of windows) with the right frequency
- external vibration at tremor frequency is rejected (<5% false tremor)
- tremor score grows with amplitude
- WFLC locks onto the tremor frequency and removes >70% of it
- EMG median frequency falls for fatigued (low-passed) EMG

> **Validation status:** thresholds are tuned and tested on synthetic data and bench recordings by the team. Clinical validation with people with tremor is the next milestone.

## Hardware

ESP32-C6 (or ESP32) · MPU6050 6-axis IMU · surface EMG front-end (Upside Down Labs NPG Lite / BioAmp) with dry or gel electrodes · Li-Po battery · compression sleeve.

## License

MIT — see [LICENSE](LICENSE).

## References

1. Louis ED, McCreary M. How common is essential tremor? *Tremor Other Hyperkinet Mov* 2021;11:28.
2. Riviere CN, Rader RS, Thakor NV. Adaptive cancelling of physiological tremor for improved precision in microsurgery. *IEEE Trans Biomed Eng* 1998;45(7):839–846.
3. Farashi S et al. Effect of exercise on Parkinson's disease tremor: a meta-analysis. *Tremor Other Hyperkinet Mov* 2021;11:15.
