# Understanding a Real IoT Sensor Stream: Shower Flow Meter (WEUSEDTO)

A step-by-step data analysis of a real IoT sensor: a shower flow meter from the public WEUSEDTO residential water end-use data set. The notebook goes **from simple to detailed**: what the data looks like, how a time series is checked, when and how the sensor is used, how predictable the signal is, and whether other fixtures in the same apartment behave in the same way.

Every section separates what was **measured** from what is **assumed or inferred**, and ends with the assumptions and limitations of the analysis.

## What the analysis shows

| # | Finding (measured) |
|---|---|
| 1 | **Not a regular time series.** The logger writes a reading every 1–2 s while water flows and about every 300 s while idle (45.4% and 53.3% of all intervals; only 1.3% are anything else). |
| 2 | **Idle almost all the time.** The value is zero in 53.9% of the rows but in 98.75% of the *time*, because readings are dense while water flows. |
| 3 | **Few long events hold most of the data.** 903 flow events: 571 micro (< 10 s), 30 medium, 302 sustained (≥ 60 s). The sustained events hold 98.4% of all flowing readings and start mostly at 07–09 h and 16–19 h (local time, assumed). |
| 4 | **Each reading is close to the previous one.** Lag-1 autocorrelation is 0.992; the guess "same as the previous value" is exactly right 79.5% of the time; entropy falls from 3.50 bit (value alone) to 0.88 bit (value given the previous one). |
| 5 | **The values sit on a fixed step grid.** All 149 distinct values fit `value = floor(2.25 × k)` with k = 0…170, which matches the vendor specification of the sensor type (about 2.25 ml per pulse). |
| 6 | **Flow levels differ between events.** The spread of event means (23.6 ml/s) is as large as the spread inside an event (21.2 ml/s); 94.3% of the steps inside events are within ±1 pulse. |
| 7 | **Time follows the value.** After a zero reading the next gap is 300 s or more in 99.2% of the cases; after a non-zero reading it is shorter in 98.9%. |
| 8 | **Other fixtures show the same pattern with different difficulty.** They are idle 81–99% of the time and the entropy falls 47–75% once the previous value is known; the washing machine is the hardest to predict. |

The notebook also contains checks on the sensor working range and the tail of the distribution, event sizes and recording quality inside events, drift by month, a constant check for the step grid, shared data gaps across fixtures, and the use of each fixture. Read the printed tables and plots there.

## Selected plots

Run the notebook to create these (they are saved in `figures/`):

| | |
|---|---|
| `plot0_first_readings.png` | The first readings by reading number and by real time |
| `plot3_intervals.png` | Time between two readings |
| `plot_event_sizes.png` | Duration, volume and mean flow of the long events |
| `plot_lag.png` | Previous value vs current value |
| `plot_entropy_ladder.png` | How hard is it to guess a reading? |
| `plot_fixture_fingerprint.png` | Comparison of the fixtures |

## Notebook structure

| Part | Sections | Question |
|---|---|---|
| **A. Looking at the data** | 1–5 | What is in the file? How is a time series checked (size, missing values, regularity, gaps)? What do the values and the flow events look like? When is water used? |
| **B. Looking deeper** | 6–9 | How predictable is the signal (autocorrelation, entropy)? How are the values built (step grid, pulse counting)? How large is the noise inside events? How does time behave (heartbeat)? |
| **C. Other fixtures and summary** | 10–11 | Do the findings hold for the other fixtures? What are the overall findings, assumptions and limitations? |

## Data

- **Data set:** WEUSEDTO, residential water end-use data from one apartment, licence **CC BY 4.0**.
- **Repositories:** [Water-End-Use-Dataset-Tools/WEUSEDTO](https://github.com/Water-End-Use-Dataset-Tools/WEUSEDTO) and [AnnaDiMauro/WEUSEDTO-Data](https://github.com/AnnaDiMauro/WEUSEDTO-Data).
- **Citation:** please cite the dataset publication listed in the README of the dataset repository.
- **Not included here.** The data files are not part of this repository; download them as described below.

## How to run

1. Download the `WEUSEDTO-Data` repository as a ZIP file and unzip it next to the notebook. The folder name starts with `WEUSEDTO-Data-`.
2. Install the requirements:
   ```bash
   pip install pandas numpy matplotlib jupyter
   ```
3. Open `Water_meter_clean.ipynb` and use **Restart & Run All**.

Expected layout:

```text
.
├── Water_meter_clean.ipynb
├── WEUSEDTO-Data-<commit>/      # downloaded, not part of this repository
│   └── Dataset/
│       ├── feed_Shower.MYD.csv
│       └── ...
├── figures/                     # created by the notebook
└── outputs/                     # created by the notebook
```

Files written to `outputs/`: `profile_shower.json` (all results of the shower analysis), `fingerprint_shower.csv`, `events_shower.csv`, `fingerprint_fixtures.csv` and `behaviour_fixtures.csv`.

## Data issues found

- **Washbasin:** one row has an impossible timestamp (time jumps back by about 50 years); it is removed by a time filter.
- **Kitchen faucet and bidet:** a different file format (comma-separated with a `Time,Flow` header); detected automatically.
- **Whole house:** the values are huge and alternate in sign, so the file cannot be read as water flow; it is excluded from all comparisons (its timestamps are only used in the coverage plot). The cause is unknown.
- **Shower:** only 358 of 629 calendar days contain data; the longest gap is 247.4 days (2019-10-28 to 2020-07-02).

## Assumptions and limitations

**Assumptions (not verified)**
- The value is flow in ml/s, as stated in the dataset publication.
- Local time is Europe/Rome (the apartment is assumed to be in Naples).
- Valid Unix times lie between 1.5e9 and 1.7e9; this removes the one broken Washbasin row.
- The zero share by time assumes each value holds until the next reading and ignores gaps over one hour.
- Event definition: value > 0, split at gaps over 60 s; volume = flow × time to the next reading, capped at 5 s (approximate).
- The sensor specification used for comparison (about 2.25 ml per pulse, working range 1–30 L/min, accuracy about ±10%) is the vendor specification of the YF-S201 type. That the sensors in this data set are of that type should be confirmed in the dataset documentation.

**Inferred from the data, not documented**
- The logger writes a heartbeat about every 300 s while idle.
- The values are pulse counts multiplied by 2.25 and rounded down; the counting window and the formula are unverified.
- Sustained events are showers; micro events are drips, residual water or readings below the sensor's specified range.

**Limitations**
- One apartment and one measurement system; the fixtures are not independent sensors. No vibration or other sensor type was analysed.
- Single-row events have an unknown duration; the cause of the 247-day gap is unknown.
- Entropy estimates are in-sample and probably slightly optimistic.
- The "long event" threshold (60 s) was chosen for the shower and may not suit other fixtures.

## Tools

Python (pandas, NumPy, Matplotlib), Jupyter Notebook.

## Author

Fashli Adli Wal Ikhsan · [github.com/fashliadli](https://github.com/fashliadli)
