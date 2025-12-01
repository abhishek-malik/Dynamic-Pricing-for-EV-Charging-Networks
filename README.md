# EV Charging Network Analysis: Demand Forecasting & Dynamic Pricing

## 📖 Overview

This project provides a granular analysis of public Electric Vehicle (EV) charging infrastructure, moving beyond simple utilization metrics to address the complexities of grid integration and economic viability. Unlike predictable residential charging, the public municipal network operates as a **federated ecosystem** influenced by commuter schedules, weather events, and points of interest.

The core objective is to transition from reactive monitoring to **predictive modeling**, enabling operators to mitigate grid stress (the "Duck Curve") and implement dynamic pricing strategies that incentivize beneficial user behaviors.

## ⚡ Key Challenges

### 1\. The Grid Integration "Duck Curve"

A central theme in this analysis is the timing mismatch between renewable energy generation and peak demand.

  * **Beneficial Load:** Overnight sessions (e.g., at *JGU - JEROME GUN HILL ROAD*) utilize off-peak base load generation.
  * **Peak Load:** Daytime sessions in commercial districts exacerbate peak demand.
  * **Goal:** Forecast demand peaks to enable **Smart Charging** (capping rates) or load shifting via pricing.

### 2\. Economic Asset Utilization

The distinction between `charge_duration_min` (active energy transfer) and `connected_duration_min` (total time plugged in) is critical.

  * **Inefficiency:** Data reveals instances of vehicles connected for 12+ hours to draw only 41 kWh (4 hours of charging), effectively blocking the asset for 8 hours.
  * **Solution:** Calculate an **Efficiency Ratio** to identify bottlenecks where idle fees or valet services could unlock latent capacity.

-----

## 🔍 Data Ecosystem & Audit

The dataset (`EV_Charging_Data_Enhanced.csv`) presents a rich but noisy tableau of charging activities requiring rigorous auditing.

### Geospatial Heterogeneity & Climate Impact

A critical anomaly was detected in the station mix:

  * **New York City (NYC):** Humid continental climate (e.g., *Bronx, Zip 10467*). Winter temps reduce battery efficiency.
  * **St. George, Utah:** Desert climate (e.g., *Zip 84770*). Summer temps $>38^\circ C$ require active thermal management (cooling), increasing grid load.

**Implication:** Training a single model on mixed climatic zones confuses the algorithm (e.g., learning that "high heat = high efficiency" from Utah data, which is false for NYC). Spatial segregation is a prerequisite for accurate modeling.

### Temporal Complexity

  * **Session Spanning:** Sessions starting at 10:00 PM and ending at 7:00 AM span two calendar days. Naive aggregation assigns all load to the first day, creating artificial evening peaks.
  * **Correction:** Resampling methodology expands these rows into hourly buckets to accurately attribute load to 3:00 AM grid conditions.

<img width="1022" height="555" alt="image" src="https://github.com/user-attachments/assets/31afe19b-af8e-4f38-b7e0-e2c895a9a346" />


-----

## ⚙️ Methodology

### 1\. Feature Engineering

Machine learning models do not natively understand "time" or "seasonality." We project temporal dynamics into feature space:

  * **Lag Features (Autoregression):** Capturing persistence and seasonality by engineering features for demand at $t-1$ (1 hour ago) and $t-24$ (yesterday).
  * **Cyclical Time Encoding:** Mapping linear hours (0-23) to a circular feature space to preserve continuity (ensuring 23:00 is "close" to 00:00).
    $$Hour_{sin} = \sin\left(2\pi \times \frac{hour}{24}\right)$$
    $$Hour_{cos} = \cos\left(2\pi \times \frac{hour}{24}\right)$$
  * **Normalization:** Using `MinMaxScaler` to project features (Temperature, Energy) into a $[0, 1]$ interval to assist Gradient Descent convergence.

### 2\. Modeling: XGBoost (Extreme Gradient Boosting)

We employ **XGBoost**, an ensemble technique that builds Decision Trees sequentially.

  * **Why XGBoost?** It excels at handling exogenous regressors. The model splits data based on conditions (e.g., "Is it a weekday?" $\rightarrow$ "Is Temp $> 90^\circ F$?") to capture non-linear interactions like cooling load demand.
  * **Approach:** Each iteration predicts the residuals (errors) of the previous model, incrementally reducing bias.

FINAL EVALUATION METRICS
RMSE: 12.1055 kWh 
MAE:  8.8075 kWh 
R^2:  0.3167 

<img width="1277" height="543" alt="image" src="https://github.com/user-attachments/assets/09d378e7-7e6f-4c3c-ad50-4d797c68365e" />


-----

## 💲 Economic Application: Dynamic Pricing

The project implements a **Demand-Based Pricing Model** where price is a function of predicted utilization relative to capacity.

### The Pricing Formula

$$P_{dynamic} = P_{base} \times (1 + \alpha \times U_{predicted})$$

Where:

  * $P_{base}$: Standard cost of electricity (e.g., $\$0.25/kWh$).
  * $\alpha$: Sensitivity factor (e.g., $0.5$) controlling price aggressiveness.
  * $U_{predicted}$: The ratio of model-predicted energy to maximum station capacity.

### Implementation Example

  * **Scenario:** A station with 70 kWh max hourly capacity.
  * **Forecast:** LSTM/XGBoost model predicts **55 kWh** demand for 6:00 PM.
  * **Utilization ($U_{predicted}$):** $55 / 70 = 0.78$ ($78\%$).
  * **Dynamic Price:**
    $$0.25 \times (1 + 0.5 \times 0.78) = \$0.3475 \text{ per kWh}$$

**Strategic Outcome:** By publishing this price ahead of time, we encourage price-sensitive drivers to shift usage away from the 6:00 PM peak, smoothing the Duck Curve while capturing higher margins from inelastic demand.

-----

## 🛠️ Tech Stack

  * **Python 3.x**
  * **Pandas & NumPy:** Data manipulation and temporal resampling.
  * **XGBoost:** Gradient boosting framework for regression.
  * **Scikit-Learn:** Preprocessing (`MinMaxScaler`, `LabelEncoder`) and metrics (`MSE`).
  * **Matplotlib/Seaborn:** Visualization of efficiency ratios and load profiles.

## 🚀 Getting Started

1.  Clone the repository.
2.  Install dependencies: `pip install pandas xgboost scikit-learn matplotlib seaborn`.
3.  Ensure `EV_Charging_Data_Enhanced.csv` is in the root directory.
4.  Run the Jupyter Notebook `EV_XGBRegressor.ipynb` to execute the pipeline.
