# Rainfall and Public Transportation in Niterói

**A NetMob 2026 Data Challenge project investigating passenger demand, bus travel times, and spatial patterns of transit disruption in Niterói, Brazil.**

This project combines automated ticketing records, GPS mobility data, meteorological observations, GTFS feeds, and census information. It examines how rainfall is associated with public transportation patterns and develops an XGBoost model to predict bus trip duration.

## Research question

How is rainfall associated with passenger demand and bus trip duration in Niterói, and which routes and areas show the greatest changes?

The analysis has three complementary components:

- **Passenger demand:** compare weekday boarding volumes across rainfall conditions and bus routes.
- **Fleet operations and geography:** reconstruct trips, compare trip durations, and map delay hotspots alongside infrastructure indicators.
- **Trip duration prediction:** estimate journey duration using meteorological, temporal, spatial, and operational features.

## Main findings

- The aggregate correlation between rainfall and hourly passenger demand is weak, approximately **−0.05**. The saved aggregate Welch-test result is not significant at the 5% level.
- Route-level demand comparisons identify increases of approximately **25.5%–31.8%** on selected routes. These are exploratory findings based on uncorrected tests. BALDEADOR–CHARITAS shows a decrease of approximately **13.1%**, which requires caution because of reported data-availability concerns.
- Fleet analyses identify differences in trip duration on selected lines after Bonferroni correction. Delay maps highlight a spatial concentration in **northwestern Niterói**.
- The saved prediction notebook reports **MAE = 5.46 minutes** and **R² = 0.8404** on its chronological test split. Errors are larger for exceptionally long trips, and line ID and direction have greater model importance than rainfall.

These observations describe associations in the available sample; they do not establish causal effects of rainfall or infrastructure.

## Repository guide

The paths below are relative to the project root containing `ticket_weather/` and `fleet_study/`.

| Notebook | Purpose | Main saved outputs |
| --- | --- | --- |
| [`ticket_weather/demand_weekday.ipynb`](ticket_weather/demand_weekday.ipynb) | Weekday demand, hourly profiles, rainfall correlation, route-level Welch tests and demand changes | `demanda_tempo.pdf`, `dispersao.pdf`, `estacoes.pdf`, `estacoes.csv` |
| [`ticket_weather/mapa.ipynb`](ticket_weather/mapa.ipynb) | Map demand changes using route geometry and census layers | `mapa.pdf` |
| [`fleet_study/mobility_analisys.ipynb`](fleet_study/mobility_analisys.ipynb) | Trip reconstruction, duration and speed comparisons, route-level statistical tests | `chuva_duration.pdf`, `velocidade_linha.pdf`, `delay_linha.pdf` |
| [`fleet_study/mapa.ipynb`](fleet_study/mapa.ipynb) | Spatial density of positive duration deviations under different rainfall conditions | `mapa_rain.pdf`, `mapa_norain.pdf` |
| [`fleet_study/infrastructure_analysis.ipynb`](fleet_study/infrastructure_analysis.ipynb) | Census-based infrastructure indicators and spatial context | `mapa_infra.pdf` |
| [`fleet_study/model.ipynb`](fleet_study/model.ipynb) | XGBoost training, chronological evaluation and prediction diagnostics | `modelo.json`, `predict_atual.pdf`, `error.pdf`, `feature.pdf`, `ping.pdf` |

The notebooks contain Portuguese comments and exploratory cells; most exported research figures use English labels. The current project is a notebook-based research workflow rather than a packaged application.

## Data requirements

**The raw datasets are not included.** The notebooks expect the NetMob26 input files and auxiliary geographic data to be available locally. Obtain the data through the dataset's authorized access process and follow its terms of use.

| Input | Expected files or fields |
| --- | --- |
| Ticketing | CSV files under `ticket_data/`, including `transaction_date`, `view_type`, `route_name`, `day_type` and other fields referenced during preprocessing |
| Vehicle mobility | CSV files under `mobility_data/`, including `timestamp`, `tripId`, `id`, `lineId`, `lineName`, `direction`, `lat` and `lng`; some exploratory cells also reference `heading` |
| Weather | `auxiliar_data/meteorological_data.csv`, with `Timestamp (UTC)`, `Rain (mm)` and, in some notebooks, `Wind Speed (m/s)` |
| Route geometry | `auxiliar_data/linhas_onibus_niteroi.gpkg` for the demand map |
| Census geography | `ibge_census_data_2022/niteroi_census_borders.gpkg`, with layers `setores_censitarios_2022_ssagi` and `favelas_e_comunidades_censo2022` |
| Census infrastructure indicators | `ibge_census_data_2022/census_tracts_indicators/block_face_environment_indicators_pct.csv` |
| GTFS | The `TransNit_20260320` feed and, for the delay maps, `TransOceanico_20260315` and `TransOceanico_20260329`; the notebooks read stops, routes, trips and shapes as needed |

The infrastructure notebook also references a local `line_routes.json` file that is not included. Supply this file if running the corresponding cell, or review whether that exploratory read is needed for your analysis.

## Getting started

### 1. Set up a Python environment

From the project root:

```bash
python -m venv .venv
```

Activate it on Linux or macOS:

```bash
source .venv/bin/activate
```

On Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Install JupyterLab and the packages imported by the notebooks:

```bash
python -m pip install jupyterlab pandas numpy scipy statsmodels matplotlib seaborn geopandas pyogrio fiona shapely adjustText scikit-learn xgboost
```

This is a dependency list inferred from the source code, not a pinned reproduction environment. Exact package versions are not recorded in the project. Consult the [Jupyter installation guide](https://jupyter.org/install) for JupyterLab setup.

### 2. Configure input paths and timestamps

Several notebooks contain absolute paths from the original development machine. Replace these paths with your local dataset locations before running any cells.

The infrastructure notebook uses paths beginning with `../data/`, so placing a `data/` folder beside `fleet_study/` and `ticket_weather/` matches those references. Other notebooks still require their absolute paths to be updated.

**Check timezone alignment before merging weather and mobility or ticketing records.** The source code parses the weather column named `Timestamp (UTC)` and joins date/hour fields, but does not explicitly perform timezone conversion in the inspected notebooks. Verify whether the supplied weather file was already converted during external preprocessing; otherwise align it with local timestamps before reproducing the analysis.

### 3. Run the notebooks

Launch JupyterLab:

```bash
jupyter lab
```

Run each notebook from its first cell, with its containing directory as the working directory so that relative reads and exports resolve correctly.

A useful reading and execution order is:

1. `ticket_weather/demand_weekday.ipynb`
2. `ticket_weather/mapa.ipynb` — consumes `estacoes.csv` produced by the demand notebook; a saved copy is included.
3. `fleet_study/mobility_analisys.ipynb`
4. `fleet_study/mapa.ipynb`
5. `fleet_study/infrastructure_analysis.ipynb`
6. `fleet_study/model.ipynb`

The fleet and model notebooks reconstruct their own input tables; they do not rely on sharing an in-memory kernel with the other notebooks. Running the export cells overwrites the corresponding local output files.

## Methodology in the current code

### Passenger demand

The demand notebook selects weekdays using `day_type == 1`. It first aggregates boardings by hour for the overall correlation and Welch-test comparison, using a **2.5 mm** threshold on matched weather observations.

For the route-level analysis, each day is classified using the **maximum matched rainfall observation within that day**, rather than the sum of daily rainfall. Daily boarding counts are then compared between groups with Welch's t-test. Percentage change is calculated from the mean daily boarding volume in each group:

```text
change (%) = 100 × (mean rainy-group volume − mean lower-rainfall-group volume)
                  / mean lower-rainfall-group volume
```

Route-level demand p-values are not adjusted for multiple comparisons. The notebook also computes the absolute value of Cohen's d.

### Fleet operations and spatial analysis

Trips are reconstructed by grouping vehicle observations by trip identifier, day, vehicle and direction. The current fleet and model notebooks retain trips with:

- observed duration **greater than 5 and less than 300 minutes**;
- start-to-end Haversine distance **greater than 0.5 km**;
- endpoint-distance-based average speed **between 2 and 80 km/h**, inclusive.

Start-to-end distance is not the full distance traveled along the bus route, and observed duration is calculated between the first and last available GPS pings.

The fleet comparison matches rainfall by trip-start date and hour. Its route-level duration tests use the Mann–Whitney U test with Bonferroni correction. In `delay_linha.pdf`, the plotted quantity is the **difference between rainy-group and lower-rainfall-group median durations**, despite the original axis label referring to an average delay.

The spatial notebook uses maximum matched rainfall over a trip and defines a duration deviation relative to the median for the same line and direction. Positive deviations weight a kernel density estimate at trip-start coordinates. The resulting maps indicate spatial concentration; their density colorbars should not be interpreted as direct, exposure-normalized totals of delay minutes.

The infrastructure indicator is the mean of the census percentages for missing storm drains and missing paving. It provides descriptive spatial context rather than a validated causal explanation for delays.

### Predictive model

| Setting | Value in `model.ipynb` |
| --- | --- |
| Target | Observed trip duration in minutes |
| Features, in order | `rain`, `lineId`, `timeofday`, `dayofweek`, `start_lat`, `start_lng`, `end_lat`, `end_lng`, `direction` |
| Rain feature | Maximum matched rainfall over the trip |
| Estimator | `XGBRegressor`, with categorical support enabled |
| Tree method | `hist` |
| Maximum depth | `10` |
| Number of estimators | `100` |
| Learning rate | `0.2` |
| Random seed | `42` |
| Training split | Trip start time ≤ `2026-03-25 00:00:00` |
| Test split | Trip start time > `2026-03-25 00:00:00` |
| Sample weights | GPS ping density, assuming a 22.5-second interval and capped at 1 |

The trained model is saved in [`fleet_study/modelo.json`](fleet_study/modelo.json). For model serialization details, see the [XGBoost model IO documentation](https://xgboost.readthedocs.io/en/stable/tutorials/saving_model.html).

The included model file records XGBoost version **3.4.0**. The remaining package versions are not recorded.

The current feature construction uses rainfall over the observed trip and coordinates from its first and last GPS records. A deployment that predicts duration before departure must define how equivalent features will be available at prediction time.

## Results and reproducibility

The repository includes exported figures, a route-demand CSV, saved notebook outputs and a trained model. The metrics above are taken from those saved outputs; documenting the project does not constitute a fresh rerun against the raw data.

Interpret results with the following constraints in mind:

- Rainy observations are scarce relative to lower-rainfall observations.
- Missing observations and data quality can affect both trip reconstruction and rainfall matching; the notebooks use interpolation or forward filling in some preprocessing steps.
- Rainfall grouping differs across analyses. An hourly value, a daily maximum and a trip maximum are different quantities, even when they share the same 2.5 mm cutoff.
- Route-level demand comparisons are exploratory because p-values are uncorrected.
- The fitted model's feature importance describes predictive behavior, not causal effects.
- Exact reproduction requires the original inputs, consistent preprocessing, correct path and timezone configuration, and a compatible software environment.

## Authors

- Lucas Toshioka Tenório
- Fernanda Yumi Assonuma
- Pedro Henrique Campos Ribeiro
- Matheus Teixeira Passos Giarola
- Alan Uema Tavares

**Affiliation:** Institute of Mathematics and Statistics, University of São Paulo (IME-USP), Brazil.

**Contact:** lucastoshioka@usp.br

## Acknowledgments

We thank IME-USP for financial support to attend the conference, Professor Kelly Rosa Braghetto for her guidance and assistance with the funding process, and the NetMob conference organizers and Fluminense Federal University (UFF) for hosting us.
