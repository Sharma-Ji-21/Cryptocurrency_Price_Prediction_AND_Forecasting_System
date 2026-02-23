# Cryptocurrency Price Prediction & Forecasting System – API Design

> **Base URL:** `/api/v1`  
> All requests and responses use `application/json`.

---

## Table of Contents

1. [Health Check](#1-health-check)
2. [Get Supported Cryptocurrencies](#2-get-supported-cryptocurrencies)
3. [Generate Forecast](#3-generate-forecast)
4. [Prediction History](#4-prediction-history)
5. [Get Prediction Detail](#5-get-prediction-detail)
6. [Error Logs](#6-error-logs-admin--dev)
7. [Validation Rules](#validation-rules)
8. [HTTP Status Codes](#http-status-codes)
9. [Backend ML Flow](#backend-ml-flow)
10. [Database Schema](#database-schema)
11. [Notes](#notes)

---

## 1. Health Check

Check backend service status.

| Method | Endpoint  | Auth Required |
|--------|-----------|---------------|
| `GET`  | `/health` | No            |

### Response

```json
{
  "status": "ok",
  "timestamp": "2026-02-23T10:30:00Z"
}
```

---

## 2. Get Supported Cryptocurrencies

Returns the list of supported cryptocurrencies. Used to populate the dropdown on the frontend.

| Method | Endpoint   | Auth Required |
|--------|------------|---------------|
| `GET`  | `/cryptos` | No            |

### Response

```json
{
  "cryptos": ["Bitcoin", "Ethereum", "Ripple"]
}
```

---

## 3. Generate Forecast

Main ML endpoint. Triggered when the user clicks **Generate Forecast**. Runs the full ML pipeline and returns price predictions.

| Method | Endpoint   | Auth Required |
|--------|------------|---------------|
| `POST` | `/predict` | No            |

### Request Body

| Field        | Type      | Required | Description                          |
|--------------|-----------|----------|--------------------------------------|
| `crypto`     | `string`  | ✅ Yes   | Cryptocurrency name                  |
| `start_date` | `string`  | ✅ Yes   | Training start date (`YYYY-MM-DD`)   |
| `end_date`   | `string`  | ✅ Yes   | Training end date (`YYYY-MM-DD`)     |
| `horizon`    | `integer` | ✅ Yes   | Forecast horizon in days (`1/7/14`)  |
| `model`      | `string`  | ❌ No    | Model selector: `auto/rf/lstm/gru`   |

#### Example Request

```json
{
  "crypto": "Bitcoin",
  "start_date": "2020-01-01",
  "end_date": "2023-12-31",
  "horizon": 7,
  "model": "auto"
}
```

### Success Response `200 OK`

```json
{
  "status": "success",
  "crypto": "Bitcoin",
  "horizon": 7,
  "model_used": "LSTM",
  "predictions": [
    {
      "date": "2024-05-01",
      "predicted_price": 42100.12
    }
  ],
  "metrics": {
    "rmse": 320.12,
    "mae": 254.89,
    "mda": 0.73
  },
  "explainability": {
    "top_features": ["RSI", "MACD", "VWAP"]
  }
}
```

### Error Response `400 Bad Request`

```json
{
  "status": "error",
  "error_code": 400,
  "message": "Invalid date range"
}
```

---

## 4. Prediction History

Returns all stored predictions from PostgreSQL.

| Method | Endpoint   | Auth Required |
|--------|------------|---------------|
| `GET`  | `/history` | No            |

### Response `200 OK`

```json
{
  "history": [
    {
      "id": 12,
      "crypto": "Ethereum",
      "horizon": 7,
      "model_used": "RF",
      "rmse": 230.44,
      "mae": 198.22,
      "mda": 0.69,
      "created_at": "2026-02-20T09:00:00Z"
    }
  ]
}
```

---

## 5. Get Prediction Detail

Returns the full result for a single prediction by ID.

| Method | Endpoint                   | Auth Required |
|--------|----------------------------|---------------|
| `GET`  | `/history/{prediction_id}` | No            |

### Path Parameter

| Parameter       | Type      | Description           |
|-----------------|-----------|-----------------------|
| `prediction_id` | `integer` | ID of the prediction  |

### Response `200 OK`

```json
{
  "id": 12,
  "crypto": "Ethereum",
  "horizon": 7,
  "model_used": "RF",
  "predictions": [
    {
      "date": "2024-05-01",
      "predicted_price": 2850.75
    }
  ],
  "metrics": {
    "rmse": 230.44,
    "mae": 198.22,
    "mda": 0.69
  },
  "explainability": {
    "top_features": ["EMA_20", "RSI", "Volume"]
  },
  "created_at": "2026-02-20T09:00:00Z"
}
```

### Error Response `404 Not Found`

```json
{
  "status": "error",
  "error_code": 404,
  "message": "Prediction not found"
}
```

---

## 6. Error Logs (Admin / Dev)

View backend error logs. Intended for admin and developer use only.

| Method | Endpoint  | Auth Required |
|--------|-----------|---------------|
| `GET`  | `/errors` | No            |

### Response `200 OK`

```json
{
  "errors": [
    {
      "id": 1,
      "message": "Model inference failed",
      "created_at": "2026-02-22T14:00:00Z"
    }
  ]
}
```

---

## Validation Rules

| Field        | Rule                                    |
|--------------|-----------------------------------------|
| `crypto`     | Must be a supported cryptocurrency      |
| `start_date` | Valid ISO 8601 date (`YYYY-MM-DD`)      |
| `end_date`   | Must be after `start_date`              |
| `horizon`    | Must be `1`, `7`, or `14`              |
| `model`      | Must be `auto`, `rf`, `lstm`, or `gru` |

---

## HTTP Status Codes

| Code  | Meaning               |
|-------|-----------------------|
| `200` | Success               |
| `400` | Bad request           |
| `404` | Data not found        |
| `422` | Validation error      |
| `500` | Internal server error |

---

## Backend ML Flow

```
POST /predict
    │
    ├── Validate input
    ├── Fetch OHLCV data
    ├── Clean + preprocess
    ├── Feature engineering
    ├── Scaling
    ├── Sliding window creation
    ├── Load models
    ├── Run inference
    ├── Calculate RMSE / MAE / MDA
    ├── Store result in PostgreSQL
    └── Return JSON response
```

> **Fallback:** If deep learning inference fails, the baseline (RF) model is used automatically.

---

## Database Schema

### `predictions`

| Column       | Type        | Description                        |
|--------------|-------------|------------------------------------|
| `id`         | `serial`    | Primary key                        |
| `crypto`     | `varchar`   | Cryptocurrency name                |
| `horizon`    | `int`       | Forecast horizon in days           |
| `model_used` | `varchar`   | Model that generated the forecast  |
| `predictions`| `jsonb`     | Array of date/predicted_price pairs|
| `metrics`    | `jsonb`     | RMSE, MAE, MDA values              |
| `created_at` | `timestamp` | Record creation timestamp          |

### `error_logs`

| Column       | Type        | Description               |
|--------------|-------------|---------------------------|
| `id`         | `serial`    | Primary key               |
| `message`    | `text`      | Error message             |
| `created_at` | `timestamp` | Error occurrence timestamp|

---

## Notes

- Both **Web** and **Mobile** clients consume the same API endpoints.
- The **baseline (RF) model** is used automatically if deep learning (LSTM/GRU) inference fails.
- All predictions are persisted to PostgreSQL and accessible via `/history`.
- **API latency target:** < 2 seconds per request.
- Dates must be in `YYYY-MM-DD` (ISO 8601) format throughout.
