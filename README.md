# Building Occupancy & Operational Schedule Intelligence Service

**Technical Description**
Municipality of Athens

---

## 1. Service Description

This service provides the Municipality of Athens with building-level electricity load forecasts at operational time horizons spanning **fifteen minutes to twenty-four hours ahead**, generating the short-term consumption predictions that several other services in the portfolio depend on as direct inputs.

Accurate short-term forecasting is not a standalone analytical objective in this programme but a **foundational capability**: the quality of demand response scheduling, the reliability of PV self-consumption optimisation and the sensitivity of anomaly detection baselines are all directly conditioned on the accuracy of the load forecasts this service produces.

The service is therefore designed as a **shared forecasting infrastructure layer** rather than an end-user service, with its outputs consumed programmatically by other services and its performance measured by the downstream operational value it enables rather than by forecast accuracy in isolation.

---

## 2. Operational Context

The service covers every smart-metered building in the municipal portfolio, maintaining a dedicated forecasting model per building trained on that building's own consumption history and calibrated to its specific contextual drivers. The estate is highly heterogeneous in its load characteristics:

- **Administrative buildings** follow structured weekday patterns anchored to working hours — consumption rising sharply at opening, peaking through mid-morning HVAC activation and declining steadily through the afternoon
- **School buildings** exhibit term-driven on-off behaviour with sharp intra-day load ramps at opening time and pronounced sensitivity to outdoor temperature, given the age and thermal characteristics of most school fabric
- **Sports and cultural facilities** present irregular, event-driven profiles where scheduled activities drive large short-duration load steps that are difficult to anticipate from schedule data alone
- **Community centres and neighbourhood service buildings** sit between these extremes, with moderate regularity interrupted by occasional event-driven spikes

Forecast model architecture, feature selection and training window length must therefore be adapted to each building type rather than standardised across the fleet. The service manages this through a **model governance framework** that assigns each building to a behavioural category, applies the appropriate modelling strategy for that category and monitors performance continuously to detect when a building's behaviour has shifted sufficiently to warrant recategorisation or retraining.

**Weather** is the primary external driver of consumption for most buildings, with outdoor temperature explaining the largest share of day-to-day variation in thermally sensitive buildings. **Occupancy and calendar features** — the Greek public holiday schedule, school term dates and building-specific operating hours — are incorporated as categorical inputs capturing the discrete load shifts associated with occupancy state transitions.

---

## 3. Methodology and Technical Approach

### Model Training

Each building-level model is trained on a **rolling historical window of up to twelve months** of 15-minute consumption observations, updated continuously as new meter readings arrive and validated data is confirmed by the quality control layer. The rolling window keeps models sensitive to recent behavioural changes while retaining sufficient history to capture seasonal patterns and their interaction with weather variability across the full annual cycle.

For buildings with shorter histories, a **transfer learning** approach initialises the model with parameters derived from buildings of the same category and floor area before fine-tuning on site-specific data, accelerating convergence without sacrificing building-level specificity.

### Probabilistic Forecasting

Forecast uncertainty is quantified explicitly for each prediction interval rather than producing point estimates alone. Probabilistic forecasts are generated as **prediction intervals at defined confidence levels**, reflecting the combined uncertainty from weather forecast error, occupancy variability and model approximation.

These uncertainty estimates are consumed by the demand response optimisation service to bound the reliability of flexibility commitments before curtailment recommendations are issued and by the anomaly detection service to set adaptive detection thresholds accounting for the inherent unpredictability of each building at each time of day. A point forecast with no uncertainty estimate would force consuming services to apply arbitrary fixed thresholds — simultaneously too tight for high-variability buildings and too loose for stable ones — degrading the operational quality of both demand response scheduling and anomaly detection.

### Continuous Performance Monitoring

Model performance is monitored continuously against incoming actuals through an automated tracking layer computing rolling accuracy metrics per building, horizon and time-of-day stratum. Sustained degradation beyond defined thresholds triggers an **automated retraining cycle** using the most recent available data.

Operational changes expected to alter a building's consumption behaviour — schedule modifications, system replacements, occupancy regime changes, addition of new metered loads — are flagged by the central configuration management layer, triggering an **accelerated recalibration** that prevents the existing model from generating systematically biased forecasts during the transition period.

### Data Quality Handling

Meter gaps, transmission delays, implausible readings and meter reset events are handled at ingestion and isolated from the training data and forecast input pipelines. When a gap affects a forecast input feature, a **gap-filling procedure** calibrated to the building's typical behaviour under similar contextual conditions is applied, with the imputation flagged in the output metadata so consuming services can apply appropriate caution to forecasts generated under data-degraded conditions.

---

## 4. Service Outputs

Per building and forecast cycle:

- A time series of predicted consumption values at 15-minute resolution covering the full horizon from fifteen minutes to twenty-four hours ahead
- Upper and lower prediction interval bounds at defined confidence levels
- Labels: building identifier, forecast generation timestamp, model version, training data window, input feature set and data quality flags indicating any imputed inputs

For the PV-equipped buildings:

- A parallel **net consumption forecast** alongside the gross load forecast, combining the building load prediction with the generation forecast to give the actual grid draw estimate required by the demand response optimisation and PV self-consumption optimisation services

Portfolio level:

- **Aggregate forecasts** computed by summing building-level predictions across the full estate and defined sub-groups, providing the total demand outlook for portfolio flexibility assessment
- A **model performance dashboard** per building, reporting rolling accuracy metrics stratified by forecast horizon, time of day and season, alongside retraining event logs and data quality incident records — the primary diagnostic tool for investigating forecast quality issues raised by consuming services

---

## 5. Evaluation and Validation

Forecast accuracy is evaluated through a **continuous walk-forward protocol**: each forecast record is assessed against the actual consumption observed in the corresponding interval, using exclusively the information available at forecast generation time.

| Indicator | Description |
|---|---|
| **MAE / MAPE** | Computed per building, stratified by forecast horizon, building category and time of day. |
| **Probabilistic calibration** | Verification that stated prediction intervals contain the true outcome at the specified confidence rate across a sufficiently large evaluation sample; systematic miscalibration triggers recalibration of the uncertainty estimation component. |
| **Portfolio-level accuracy** | Evaluated separately from building-level accuracy, since aggregation effects can substantially reduce portfolio-level error and the consumers of portfolio-level forecasts have different accuracy requirements. |

Forecast performance results are included in the service's contribution to the intermediate and final deliverable reports, with building-level accuracy breakdowns supporting identification of building categories or time periods where forecast quality falls short of downstream operational requirements.

---

## 6. Implementation and Availability

The service is exposed through an **authenticated REST API** delivering building-level and portfolio-level load forecasts, net consumption forecasts for PV buildings, prediction intervals and model performance summaries in structured JSON format, with a **push notification mechanism** informing consuming services when updated forecasts are available for a given building or horizon window.

A **subscription model** allows each consuming service to register interest in specific buildings, building groups or horizon ranges, receiving targeted updates without polling the full forecast catalogue.

The forecasting pipeline is structured as independent versioned modules:

1. Data ingestion and quality control
2. Feature engineering
3. Model training and updating
4. Probabilistic forecast generation
5. Portfolio aggregation
6. Performance monitoring

Each module is logged and versioned independently, allowing targeted updates to individual components without disrupting the live forecast stream.
