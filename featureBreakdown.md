# 🚀 Feature Breakdown & Implementation Strategy

This section explains how each core feature of the Cryptocurrency Price Prediction & Forecasting System is implemented across the frontend, backend, ML pipeline, and database.

---

### 1. Select Cryptocurrency and Time Range

**What it does:**
Allows users to select a cryptocurrency, historical date range, forecast horizon, and model type from the Web (React) or Mobile (Flutter) UI.

**How it's implemented:**
- Frontend provides dropdowns and date pickers.
- Supported cryptocurrencies are fetched using `GET /api/v1/cryptos`.
- On clicking **Generate Forecast**, user inputs are validated and sent to backend via `POST /api/v1/predict`.
- Validation includes: valid crypto, valid date range, horizon = 1 / 7 / 14, model = auto / rf / lstm / gru.

---

### 2. Automatic Historical OHLCV Data Fetching

**What it does:**
Fetches historical Open, High, Low, Close, Volume data for the selected cryptocurrency.

**How it's implemented:**
- Backend uses Kaggle dataset / public OHLCV APIs.
- Data is fetched dynamically inside `/predict`.
- Raw data is converted to a Pandas DataFrame.
- Missing values are handled and timestamps are sorted.

```
API Request → Fetch OHLCV → Clean Data
```

---

### 3. Time-Series Preprocessing & Feature Engineering

**What it does:**
Transforms raw price data into ML-ready features.

**How it's implemented (Pandas + NumPy):**

| Feature             | Description                           |
|---------------------|---------------------------------------|
| SMA / EMA           | Simple & Exponential Moving Average   |
| RSI                 | Relative Strength Index               |
| MACD                | Moving Avg Convergence Divergence     |
| Lagged Close Prices | Previous N-day close values           |
| Rolling Mean & Std  | Window-based statistics               |
| Log Returns         | Log-scale price change                |
| VWAP                | Volume Weighted Average Price         |

Then: Scaling using pre-fitted scaler → Sliding window sequence creation for LSTM / GRU.

```
Clean Data → Feature Engineering → Scaling → Sliding Window
```

---

### 4. Train Forecasting Models

**What it does:**
Trains multiple models for price prediction.

**How it's implemented:**

| Type          | Models                           |
|---------------|----------------------------------|
| Baseline      | Linear Regression, Random Forest |
| Deep Learning | LSTM (PyTorch), GRU (PyTorch)    |

- Training happens **offline**. Models are saved and loaded at inference time.
- At runtime: selected model (or `auto`) is loaded.
- If DL inference fails → system automatically falls back to Random Forest.

---

### 5. Single-Step and Short-Horizon Prediction

**What it does:**
Generates 1-day, 7-day, or 14-day price forecasts.

**How it's implemented:**
- Sliding window sequences are passed to the trained model.
- Output prices are generated iteratively for multi-step horizons.
- Results are formatted as a JSON time series and returned via `POST /api/v1/predict`.

---

### 6. Interactive Price Chart with Predicted Values

**What it does:**
Displays historical and predicted prices on an interactive chart.

**How it's implemented:**
- React uses **Recharts** for Web.
- Flutter uses chart widgets for Mobile.
- Frontend plots Historical Close Prices and Future Predicted Prices on the same chart.

---

### 7. REST API for Prediction Requests

**What it does:**
Exposes the full ML pipeline through REST endpoints.

**How it's implemented — FastAPI endpoints:**

| Method | Endpoint          | Description                    |
|--------|-------------------|--------------------------------|
| GET    | `/health`         | Backend status check           |
| GET    | `/cryptos`        | List supported cryptocurrencies|
| POST   | `/predict`        | Run ML pipeline, return forecast|
| GET    | `/history`        | Fetch all stored predictions   |
| GET    | `/history/{id}`   | Fetch single prediction by ID  |
| GET    | `/errors`         | View backend error logs        |

---

### 8. Save Predictions & Experiments to Database

**What it does:**
Persists all forecasts, metrics, and explainability for the history view.

**How it's implemented:**
Each `POST /predict` call writes to:

| Table                 | Data Stored                                  |
|-----------------------|----------------------------------------------|
| `prediction_requests` | User inputs, crypto, horizon, model selected |
| `predictions`         | Predicted price time series (JSONB)          |
| `prediction_metrics`  | RMSE, MAE, MDA + top features (JSONB)        |

History is retrieved via `GET /api/v1/history`.

---

### 9. Actual vs Predicted Comparison

**What it does:**
Compares model output against real prices to evaluate performance.

**How it's implemented:**
- Backend computes RMSE, MAE, and MDA.
- Actual vs Predicted series generated side-by-side.
- Residual error distribution calculated.
- Frontend renders comparison charts for visual evaluation.

---

## 🔁 End-to-End Flow

```
User Input
    → API Request (POST /predict)
    → OHLCV Fetch
    → Feature Engineering
    → Model Inference
    → Metrics Calculation
    → Save to PostgreSQL
    → Return JSON
    → Charts + Metrics Display
```

---

## ✅ Reliability Features

| Feature             | Description                                          |
|---------------------|------------------------------------------------------|
| Input Validation    | All fields validated before ML pipeline triggers     |
| Loading States      | UI shows indicator during inference                  |
| Error Handling      | All failures logged to `error_logs`                  |
| Model Fallback      | DL failure → auto fallback to Random Forest          |
| Prediction History  | Every forecast saved and retrievable via `/history`  |
| Unified API         | Same endpoints used by both Web and Mobile           |
