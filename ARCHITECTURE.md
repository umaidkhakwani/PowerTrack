# PowerTrack - Technical Architecture & Logic Documentation

This document provides a deep dive into the project structure, file responsibilities, and the underlying logic of the PowerTrack application.

## 📂 Project Structure Overview

```text
PowerTrack/
├── backend/                  # NestJS API Server (Port 3000)
│   ├── src/
│   │   ├── auth/             # Authentication Module (JWT, Login, Register)
│   │   ├── common/           # Shared Code (Filters, Guards)
│   │   ├── consumption/      # Consumption Logic (CRUD, Export, Analytics Proxy)
│   │   ├── properties/       # Property Management
│   │   ├── users/            # User Management
│   │   ├── main.ts           # App Entry Point (Swagger, Global Pipes)
│   │   └── seed-dataset.ts   # Data Ingestion Script
│   └── analytics-service/    # Python Math Service (Port 8000)
│       └── main.py           # Linear Regression & Z-Score Logic
├── frontend/                 # React + Vite Client (Port 5173)
│   ├── src/
│   │   ├── pages/            # Dashboard.tsx, PropertyDetails.tsx
│   │   ├── components/       # Reusable UI (Forms, Layouts)
│   │   └── api/              # Axios Client
└── dataset/                  # Raw Data Files
```

---

## 🏗️ Backend (NestJS)

The backend is built with **NestJS**, a modular framework for Node.js. It follows a **Controller-Service-Repository** pattern.

### 1. Consumption Module
**Key Files:**
*   `consumption.controller.ts`: Defines API endpoints (`GET /`, `POST /export`, `POST /analytics/...`). Handles Request/Response and DTO validation.
*   `consumption.service.ts`: Contains the business logic and database queries.
*   `consumption.module.ts`: Wires up the controller, service, and dependencies (HttpModule).

**Key Logic:**
*   **Smart Aggregation**:
    *   **Goal**: Prevent frontend crashes when loading millions of rows.
    *   **Logic**: The `findAll` method accepts a `resolution` parameter ('raw', 'day', 'month'). It uses PostgreSQL's `date_trunc` function to group data *in the database* before sending it to the client.
    *   *Example*: If you request "This Year", the backend groups data by 'month' and returns only 12 rows instead of 100,000+.
*   **Analytics Proxy**:
    *   The backend acts as a secure gateway. It fetches raw data from the DB, formats it, and sends it to the Python service via HTTP (Axios).
*   **Robust Error Handling**:
    *   `http-exception.filter.ts`: A global filter that catches all crashes and formats them into JSON (`{ statusCode, message, timestamp }`).

### 2. Authentication Module
**Key Files:**
*   `auth.service.ts`, `jwt.strategy.ts`
**Logic**:
*   Uses **Passport.js** and **JWT (JSON Web Tokens)**.
*   When a user logs in, they receive a signed JWT. This token must be sent in the `Authorization: Bearer <token>` header for all subsequent requests.

---

## 🐍 Analytics Service (Python)

A microservice built with **FastAPI** to handle heavy mathematical computations.

**File**: `backend/analytics-service/main.py`

### 1. Trend Analysis
**Endpoint**: `/analytics/trend`
**Logic**:
*   **Input**: List of `{ date, value }`.
*   **Algorithm**: **Linear Regression (OLS)**.
*   It fits a line (`y = mx + c`) through the data points.
*   **Slope (`m`)**:
    *   If `m > 0.01`: Trend is **Increasing**.
    *   If `m < -0.01`: Trend is **Decreasing**.
    *   Otherwise: Trend is **Stable**.
*   **Confidence**: Calculated using R-Squared value (0-1).

### 2. Anomaly Detection
**Endpoint**: `/analytics/detect-anomalies`
**Logic**:
*   **Goal**: Detect if the *latest* data point is abnormal compared to recent history.
*   **Algorithm**: **Z-Score / Standard Deviation**.
    1.  Take the filtering range (e.g., "This Month").
    2.  Isolate the **last 5 data points** as the "Reference Window".
    3.  Calculate **Mean** ($\mu$) and **Standard Deviation** ($\sigma$) of this window.
    4.  Calculate **Limit** = $\mu + 2\sigma$.
    5.  **Check**: If Latest Value > Limit, it is an **Anomaly**.

---

## 💻 Frontend (React)

### 1. Property Details Page
**File**: `src/pages/PropertyDetails.tsx`
**Features**:
*   **Dynamic Graph**: Uses `Recharts`.
    *   *Logic*: Automatically requests different resolutions ('day', 'month') based on the `filterTime` state to keep the UI snappy.
*   **Analytics Integration**:
    *   Sends *filters* to the backend -> Backend fetches data -> Python analyzes -> Frontend displays result.
    *   Shows clear messages: "Latest reading is normal. Value < Limit".
*   **Export Modal**:
    *   Allows users to download CSVs. The actual CSV generation happens on the Backend (`consumption.service.ts -> exportToCsv`) to stream data efficiently.

---

## 🔄 Data Flow Example: "Detect Anomalies"

1.  **User** clicks "Detect Anomalies" on Frontend.
2.  **Frontend** sends filters (`{ from: '2023-01-01', to: '2023-01-31' }`) to Backend.
3.  **Backend** (`ConsumptionController`):
    *   Calls `ConsumptionService.findAll` to get the actual data rows from PostgreSQL.
    *   Calls `ConsumptionService.analyzeSpike` with this data.
4.  **Backend** forwards the data to Python (`http://analytics:8000/analytics/spike`).
5.  **Python**:
    *   Calculates Mean and Limit based on the last 5 days.
    *   Returns result (`is_spike: true/false`).
6.  **Backend**: Returns result to Frontend.
7.  **Frontend**: Displays "Anomaly Detected" or "Normal".
