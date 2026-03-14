## Task 1: Pipeline Architecture Diagram

### 1.1 — Draw the end-to-end architecture

```text
[ SOURCE LAYER ]          [ INGESTION LAYER ]       [ STORAGE LAYER ]          [ CONSUMER LAYER ]
                                                                             
Batch File (XLSX) ────┐                               ┌──────────────┐         ┌────────────────┐
                      │   ┌──────────────┐            │ Raw Layer    │         │ BI Dashboard   │
                      ├──►│ Ingestion    │───────────►│ (Parquet)    │────┐    │ (Sales/Trends) │
                      │   │ (Validation) │            └──────────────┘    │    └────────────────┘
Live Stream (Row) ────┘   └──────┬───────┘                    │           │
                                 │                            ▼           │
                                 │                    ┌──────────────┐    │    ┌────────────────┐
                                 │                    │ Clean Layer  │────┴───►│ ML Model       │
                          ┌──────▼───────┐            │ (Parquet)    │         │ (High-Value)   │
                          │ Quarantine / │            └──────────────┘         └────────────────┘
                          │ Dead Letter  │                    │
                          │ (JSON/CSV)   │                    ▼
                          └──────────────┘            ┌──────────────┐
                                                      │ Feature Layer│
                                                      │ (Avro/DB)    │
                                                      └──────────────┘
                                                              ▲
                                                              │
[ MONITORING ] ◄──────────────────────────────────────────────┘
(Quality, Drift, Freshness)

```

---

### 1.2 — Describe each component 

* **Ingestion Layer (Validation):** This is the entry point for both the historical Excel file and the live stream. It acts as a gatekeeper, performing schema and business rule checks in real-time. Valid data is passed to Raw storage, while failures are diverted to the Quarantine area.
* **Raw Layer (Storage):** This is an immutable landing zone where data is stored in its original form but converted to **Parquet** format for efficient storage. It is organized by `ingestion_date` to allow for full history replays if logic changes later.
* **Quarantine / Dead Letter Area:** A storage bucket specifically for records that failed validation. We store these in a human-readable format like **JSON** so engineers can inspect the errors, fix source issues, and eventually re-inject the records into the pipeline.
* **Clean Layer (Storage):** This layer contains the "Golden Records." Data here has been deduplicated, standardized (e.g., date formats), and enriched with basic derived columns like `LineTotal`. This is the primary source for the BI Dashboard.
* **Feature Layer (Storage):** A specialized layer that stores aggregated, customer-level metrics (e.g., `total_revenue`, `recency`). We use **Avro** or a specialized database here because it needs to support fast lookups for the ML model's training and prediction phases.


## Task 2: Validation and Error Handling Design

### 2.1 — Define validation rules

| Category | Field | Rule |
| --- | --- | --- |
| **Schema** | `InvoiceNo` | Must be a non-empty String. |
| **Schema** | `CustomerID` | Must be an Integer (required for Feature Layer). |
| **Schema** | `InvoiceDate` | Must follow ISO-8601 format (`YYYY-MM-DD HH:MM:SS`). |
| **Value Range** | `UnitPrice` | Must be `> 0`. |
| **Value Range** | `Quantity` | Must be between `-100,000` and `100,000`. |
| **Value Range** | `Country` | Must be within a predefined list of ISO country names. |
| **Business Rule** | Cancellation | If `InvoiceNo` starts with 'C', `Quantity` **must** be negative. |
| **Business Rule** | StockCode | Must match the pattern `\d{5}[A-Z]?` or specific service codes (e.g., 'POST'). |
| **Business Rule** | Logic | `UnitPrice * Quantity` must not exceed a total of £1,000,000 per line (fraud check). |

---

### 2.2 — Design the error handling flow

1. **Rejection & Quarantine:** Any record failing a **Schema** or **Business Rule** validation is rejected from the main flow and written to the **Quarantine area**.
2. **Alerting:** If the quarantine rate exceeds **2%** of the batch volume or **5 consecutive** streaming rows, an automated alert is sent to the Data Engineering team via Slack/Email.
3. **Recovery:** Once the cause (e.g., an upstream system change) is fixed, a "Reprocessing Script" reads from the Quarantine area, validates the data again, and pushes it back into the Ingestion Layer.


## Task 3: Transformation and Storage Design

### 3.1 — Define Transformations

| Transformation | Input | Output | Idempotency |
| --- | --- | --- | --- |
| **Line Total Calculation** | `UnitPrice`, `Quantity` | `LineTotal` column | **Yes**: Re-running the calculation $P \times Q$ always yields the same result. |
| **Standardization** | Raw `Country`, `InvoiceDate` | Cleaned `Country`, ISO-Date | **Yes**: Applying a mapping or format mask to the same input is consistent. |
| **Customer Aggregation** | Cleaned Transactions | `TotalRevenue`, `OrderCount` | **Yes**: If using a "Delete and Insert" or "Overwrite" strategy for the specific date window. |
| **Temporal Feature Engineering** | Historical features | Lagged features (e.g., "Revenue last 30 days") | **Yes**: As long as the "Observation Window" end-date is fixed. |

---

### 3.2 — Design the Storage Layers

| Layer | Contents | Format | Update Frequency | Retention |
| --- | --- | --- | --- | --- |
| **Raw** | Exact copy of source data with ingestion timestamp. | **Parquet** | As it arrives (Batch/Stream) | Permanent (for auditing) |
| **Clean** | Validated, deduplicated transactions; standardized fields. | **Parquet** | Daily (Batch) / Real-time | 5+ Years |
| **Feature** | Customer-level aggregates and ML-ready vectors. | **Avro** | Daily | Current + 1 Year History |

**Justification:** * **Parquet** is chosen for Raw and Clean layers to optimize for **columnar reads**, which is ideal for BI dashboards that aggregate sales by date or country.

* **Avro** is used for the Feature layer because its **row-based format** and schema evolution support are better suited for ML pipelines that need to read an entire customer profile at once for a prediction.

---

### 3.3 — Design Incremental Updates

To handle the continuous flow of data without reprocessing the entire history, we will implement the following:

* **High-Water Mark Tracking:** The pipeline will store the `max(InvoiceDate)` or `max(IngestionTimestamp)` from the previous run. New runs will only fetch records where `InvoiceDate > high_water_mark`.
* **Late-Arriving Records:** If a transaction from "yesterday" arrives "today," the high-water mark logic will still catch it based on its ingestion time, and the Clean layer will use an **Upsert** (Update/Insert) operation based on `InvoiceNo` to prevent duplicates.
* **Feature Refresh:** Customer-level features will be updated using a **Sliding Window**. Every night, the pipeline recalculates metrics for any customer who had activity in the last 24 hours.