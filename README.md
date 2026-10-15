# Energy-aware sampling of a river-level sensor with an EnKF and MPC

A river-level sensor that reports more often gives a flood forecast more to work with, but it also drains its battery faster. This repository holds one Jupyter notebook that measures that trade-off at USGS site 07022000 (Mississippi River at Thebes, IL) and uses it to schedule the sensor's sampling rate.

## What the notebook does

**Part 1: forecast skill vs. sampling frequency (Sections 1–9).** This is an observing-system simulation experiment. Real USGS stream levels are the truth. Synthetic sensor measurements (truth plus Gaussian noise) are assimilated by an Ensemble Kalman Filter (EnKF) into synthetic ensemble forecasts whose error grows with lead time. The EnKF runs at fixed sampling frequencies of 1, 5, 10 and 15 measurements per hour. Each forecast is scored by its hit rate (HR) and false alarm rate (FAR) for exceeding the 30 ft action stage. Fitting the scores gives closed-form skill curves in the sampling frequency $f$:

$$HR(f) = A - B\,e^{-kf}, \qquad FAR(f) = A'\,e^{-k'f} + C'$$

**Part 2: energy-aware sampling control (Section 10).** A model predictive controller (MPC) chooses the sensor's sampling frequency every hour. A cost–loss model combines the skill curves with the online EnKF's event probabilities into the expected value of sampling faster. The controller trades that value against the sensor node's state of energy, using a battery model and a solar-harvest forecast. The final figure shows the chosen frequency, the stream level and the battery level, with the EnKF's event probability for each of the next 12 hours underneath.

## Repository layout

```
.
├── enkf_mpc_adaptive_sampling.ipynb   # the notebook, with saved outputs
├── data/
│   ├── processed_mississippi_data_interpolated_2021_2022.csv
│   ├── GHI Data USGS Site.csv
│   └── voltage_vs_SoC_LUT.csv
└── outputs/
    ├── forecast_evaluation_metrics_1_R_0.25.csv
    ├── forecast_evaluation_metrics_5_R_0.25.csv
    ├── forecast_evaluation_metrics_10_R_0.25.csv
    └── forecast_evaluation_metrics_15_R_0.25.csv
```

## Data

| File | Contents |
|---|---|
| `data/processed_mississippi_data_interpolated_2021_2022.csv` | USGS stream level (ft) at site 07022000, every 30 minutes, 2021-01-01 to 2022-01-01 |
| `data/GHI Data USGS Site.csv` | NASA POWER hourly all-sky surface shortwave irradiance (Wh/m²) at 37.22° N, 89.46° W, 2021-01-01 to 2021-07-30 |
| `data/voltage_vs_SoC_LUT.csv` | Battery open-circuit voltage (V) vs state of charge (%), used by the energy model |

## Precomputed results

`outputs/forecast_evaluation_metrics_{f}_R_0.25.csv` hold precomputed Part 1 skill scores for each sampling frequency $f$, at sensor noise variance $R$ = 0.25 ft². Each file has one row per trial and alarm threshold $p_t$, with columns `trial`, `frequency`, `p_t`, `hit_rate`, `false_alarm_rate` and `status`. With them, the curve fits (Section 8) and the skill heatmap (Section 9.1) run without rerunning the EnKF.

## Getting started

Developed with Python 3.13.

```bash
git clone https://github.com/NihalAhmad322/enkf-mpc-adaptive-sampling.git
cd enkf-mpc-adaptive-sampling
pip install numpy==2.3.2 pandas==2.3.1 scipy==1.16.0 scikit-learn==1.9.0 matplotlib==3.11.2 seaborn==0.13.2 tqdm==4.68.2 pyarrow==23.0.1 jupyter
jupyter notebook enkf_mpc_adaptive_sampling.ipynb
```

The notebook uses paths relative to the repository root, so start Jupyter from there. Saving the assimilation animation (Section 9.2) also needs [ffmpeg](https://ffmpeg.org/) on your PATH.

## Running the notebook

- **Sections 1–5** load the data and define the functions and parameters. Run them first.
- **Section 6** runs the offline EnKF and writes Parquet files to `outputs/EnKF Frequency {f} Sigma {sigma} R {R} Trial {trial}/`. `t_windows` sets how many 12-hour windows are simulated from 2021-01-01. The river first reaches the 30 ft threshold on 2021-03-18, so the default of 5 windows is only a quick test: it contains no events, and every score comes out as 0. Use at least about 155 windows for meaningful scores.
- **Section 7** scores the forecasts and writes `outputs/forecast_evaluation_metrics_*.csv`, replacing the precomputed files. Running it after a short test run overwrites them with zeros.
- **Sections 8 and 9.1** fit and plot the skill curves from those CSVs.
- **Section 9.2** replays the EnKF from the Parquet files of a Section 6 run made with `viz=True`.
- **Section 10** runs the MPC + online EnKF loop. Its skill curves are hard-coded, so it does not need the CSVs. It sets its own run parameters, overwriting Section 5's, so rerun Section 5 before going back to Sections 6–9.
