## Task 1: Pipeline Architecture Diagram

### 1.1 — Draw the End-to-End Architecture

```
[ SOURCE LAYER ]
┌──────────────────────┐       ┌──────────────────────┐
│ Batch File (XLSX)    │       │ Live Stream          │
│ (Historical Load)    │       │ (Real-Time Events)   │
└──────────┬───────────┘       └──────────┬───────────┘
           │                              │
           └──────────────┬───────────────┘
                          ▼

[ INGESTION LAYER ]
                  ┌──────────────────────────┐
                  │ Ingestion Service        │
                  │ (Batch + Streaming)      │
                  └──────────┬───────────────┘
                             ▼

[ VALIDATION STAGE ]
                  ┌──────────────────────────┐
                  │ Validation Engine        │
                  │ (Schema + Business Rules)│
                  └───────┬──────────┬───────┘
                          │          │
                          │          ▼
                          │   ┌──────────────────────┐
                          │   │ Quarantine /         │
                          │   │ Dead Letter Queue    │
                          │   │ (Invalid Records)    │
                          │   └──────────────────────┘
                          ▼

[ STORAGE LAYER - RAW ]
                  ┌──────────────────────────┐
                  │ Raw Layer (Parquet)      │
                  │ Immutable Landing Zone   │
                  └──────────┬───────────────┘
                             ▼

[ TRANSFORMATION STAGE ]
                  ┌──────────────────────────┐
                  │ Transformation Engine    │
                  │ (Cleaning + Enrichment)  │
                  └──────────┬───────────────┘
                             ▼

[ STORAGE LAYER - CLEAN ]
                  ┌──────────────────────────┐
                  │ Clean Layer (Parquet)    │
                  │ Analytics-Ready Data     │
                  └──────────┬───────────────┘
                             ▼

[ STORAGE LAYER - FEATURE ]
                  ┌──────────────────────────┐
                  │ Feature Layer (Avro/DB)  │
                  │ ML Features              │
                  └──────────┬───────────────┘
                             │
          ┌──────────────────┴──────────────────┐
          ▼                                     ▼

[ CONSUMER LAYER ]
┌──────────────────────────┐       ┌──────────────────────────┐
│ BI Dashboard             │       │ ML Model                 │
│ (Sales, Trends, Reports) │       │ (High-Value Prediction)  │
│ ← Reads from Clean Layer │       │ ← Reads from Feature     │
└──────────────────────────┘       └──────────────────────────┘


[ MONITORING & OBSERVABILITY ]
┌────────────────────────────────────────────────────────────┐
│ Data Quality | Freshness | Volume | Schema Drift Monitoring│
│ Observes all stages: Ingestion, Validation, Storage,       │
│ Transformation, and Consumption                            │
└────────────────────────────────────────────────────────────┘
```

---

### 1.2 — Describe Each Component 

### **Batch File (XLSX) – Historical Source**

This component represents the full historical dataset (`Online Retail.xlsx`) that is ingested once as a batch load. It contains past transactional records used to initialize the pipeline and backfill storage layers. The input is a static file, and the output is a stream or batch of records passed to the ingestion service. This source ensures that the system has complete historical coverage before processing incremental updates.

---

### **Live Stream (Row-by-Row Transactions)**

This component simulates real-time transaction events arriving one record at a time. It represents ongoing business activity and feeds incremental updates into the pipeline. The input is individual transaction records, and the output is a continuous stream sent to the ingestion service. This ensures the pipeline supports near real-time data processing alongside batch ingestion.

---

### **Ingestion Service (Batch + Streaming)**

The ingestion service is responsible for receiving data from both batch and streaming sources and standardizing it into a unified internal format. It handles parsing (e.g., XLSX to structured records), basic schema alignment, and routing to the validation stage. The input consists of raw records from both sources, and the output is a normalized stream of records ready for validation. This layer abstracts source-specific differences and provides a consistent entry point into the pipeline.

---

### **Validation Engine (Schema + Business Rules)**

The validation engine enforces data quality by applying schema checks, value constraints, and business rules to each record. It receives normalized records from the ingestion layer and evaluates them against predefined validation logic. Valid records are passed forward into the storage layer, while invalid records are diverted to the quarantine system. This stage ensures that only high-quality, consistent data enters downstream processing.

---

### **Quarantine / Dead Letter Queue (Invalid Records)**

This component stores records that fail validation checks for further inspection and recovery. It receives invalid records along with error metadata (e.g., failed rule, timestamp) from the validation engine. The data is stored in flexible formats such as JSON or CSV for easy debugging and reprocessing. This layer ensures that bad data is not lost and can be corrected and reintroduced into the pipeline later.

---

### **Raw Layer (Parquet – Immutable Landing Zone)**

The raw layer is the first persistent storage stage for validated data and acts as an immutable record of ingested transactions. It stores data in Parquet format, partitioned by ingestion date to support efficient querying and reprocessing. The input is validated records from the validation stage, and the output is raw data used by downstream transformations. This layer enables data lineage, auditing, and recovery by preserving original records.

---

### **Transformation Engine (Cleaning + Enrichment)**

The transformation engine processes raw data into analysis-ready form by applying cleaning, normalization, and enrichment logic. It reads from the raw layer and performs operations such as handling edge cases, computing derived columns (e.g., total price), and preparing structured outputs. The output is clean, structured data written to the clean layer. This stage ensures that downstream consumers work with consistent and enriched datasets.

---

### **Clean Layer (Parquet – Analytics-Ready Data)**

The clean layer stores processed, high-quality data optimized for analytical workloads. It uses Parquet format to support efficient columnar queries required by BI tools. The input is transformed data from the transformation engine, and the output serves as the primary source for reporting and dashboards. This layer represents the trusted dataset for business intelligence use cases.

---

### **Feature Layer (Avro / Database – ML Features)**

The feature layer contains engineered features aggregated at the customer level for machine learning use cases. It receives input from the clean layer and transformation logic that computes features such as total spend, frequency, and recency. Data is stored in Avro or a database format optimized for row-based access and fast retrieval. This layer provides structured inputs for model training and inference.

---

### **BI Dashboard (Sales, Trends, Reports)**

The BI dashboard is a consumer-facing component that provides business insights through visualizations and reports. It reads data directly from the clean layer, which is optimized for analytical queries. The dashboard presents metrics such as daily sales, top products, and geographic trends. This component transforms processed data into actionable insights for decision-makers.

---

### **ML Model (High-Value Customer Prediction)**

The ML model consumes features from the feature layer to predict whether a customer is high-value. It uses structured feature inputs such as purchase frequency and total revenue to train and update predictive models. The model is retrained periodically (e.g., weekly) and uses updated features generated daily. This component enables data-driven decision-making through predictive analytics.

---

### **Monitoring & Observability (Quality, Drift, Freshness)**

This component tracks the health and reliability of the entire pipeline. It monitors metrics such as data freshness, volume consistency, schema changes, and validation failure rates. Inputs are collected from multiple stages (ingestion, validation, storage, and consumption), and outputs include alerts and logs for operators. This layer ensures early detection of issues and maintains trust in the data pipeline.


## Task 2: Validation and Error Handling Design

### 2.1 — Define Validation Rules

| Category | Field | Rule |
| :--- | :--- | :--- |
| Schema | InvoiceNo | Must be a non-empty string. |
| Schema | StockCode | Must be a non-empty string. |
| Schema | Description | Must be a string (nullable allowed). |
| Schema | Quantity | Must be an integer. |
| Schema | InvoiceDate | Must be a valid timestamp (YYYY-MM-DD HH:MM:SS). |
| Schema | UnitPrice | Must be a float. |
| Schema | CustomerID | Must be a non-null integer (required for Feature Layer). |
| Schema | Country | Must be a non-empty string. |

| Category | Field | Rule |
| :--- | :--- | :--- |
| Value Range | UnitPrice | Must be > 0 (except for valid refund adjustments). |
| Value Range | Quantity | Must not be 0. |
| Value Range | Quantity | Must be between -100,000 and 100,000. |
| Value Range | Country | Must exist in predefined ISO country list. |

| Category | Field | Rule |
| :--- | :--- | :--- |
| Business Rule | Cancellation | If InvoiceNo starts with 'C', Quantity must be negative. |
| Business Rule | Consistency | If Quantity < 0, then InvoiceNo must start with 'C'. |
| Business Rule | StockCode | Must match pattern (\d{5}[A-Z]?) OR known service codes (e.g., 'POST'). |
| Business Rule | Revenue Cap | UnitPrice × Quantity must not exceed £1,000,000 per row. |
| Business Rule | CustomerID | Transactions without CustomerID are excluded from Feature Layer but allowed in Clean Layer. |


### 2.2 — Design The Error Handling Flow

### **1. Schema Failures**

* Records failing schema validation are **rejected immediately**
* Sent to **Quarantine Layer (JSON/CSV)** with error metadata
* These are considered **hard failures**
* No downstream processing allowed

---

### **2. Value Range Failures**

* Records are **rejected and quarantined**
* Tagged with specific rule violations
* Counted toward **data quality metrics**

---

### **3. Business Rule Failures**

* Records are **quarantined but preserved**
* Some may be recoverable (e.g., incorrect sign on Quantity)
* Marked for **manual review or automated correction pipelines**

---

### **4. Alerting Strategy**

* Trigger alert if:

  * > **2%** of batch fails validation
  * OR > **5 consecutive streaming failures**
* Alerts sent via:

  * Slack / Email
* Includes:

  * Failed rule breakdown
  * Sample records

---

### **5. Recovery & Reprocessing**

* Quarantined data stored with:

  * Original payload
  * Error reason
* After fixing root cause:

  * Reprocessing job reads quarantine data
  * Revalidates records
  * Sends valid records back to **Ingestion Layer**



## Task 3: Transformation and Storage Design

### 3.1 — Define Transformations
 

| Transformation | Input | Output | Idempotency |
| :--- | :--- | :--- | :--- |
| Line Total Calculation | UnitPrice, Quantity | LineTotal | Yes |
| Cancellation Flag | InvoiceNo | IsCancellation (Boolean) | Yes |
| Date Normalization | InvoiceDate | Standardized Timestamp | Yes |
| Country Standardization | Country | Clean Country Names | Yes |
| Deduplication | Raw Records | Unique Transactions | Yes (using InvoiceNo + StockCode) |
| Customer Aggregation | Clean Transactions | TotalRevenue, OrderCount | Yes (overwrite per window) |
| Product Diversity | Clean Transactions | UniqueProductCount per Customer | Yes |
| Recency Calculation | Customer History | DaysSinceLastPurchase | Yes |
| Sliding Window Features | Clean Layer | Last 7/30-day metrics | Yes (fixed window) |


### 3.2 — Design the Storage Layers


| Layer | Contents | Format | Update Frequency | Retention |
| :--- | :--- | :--- | :--- | :--- |
| Raw | Validated records + ingestion timestamp | Parquet | Continuous (Batch + Stream) | Permanent |
| Clean | Deduplicated, standardized, enriched transactions | Parquet | Near real-time / Daily | 5+ years |
| Feature | Aggregated customer-level ML features | Avro / DB | Daily | Current + 1 year |


### Justification:

* **Raw Layer (Parquet)**
  Optimized for storage efficiency and reprocessing. Partitioned by ingestion date.

* **Clean Layer (Parquet)**
  Columnar format ideal for BI queries (aggregations, filtering).

* **Feature Layer (Avro/DB)**
  Row-based access required for ML → fast retrieval per customer.


### 3.3 — Design Incremental Updates 

### **High-Water Mark Tracking**

* Track `max(IngestionTimestamp)` instead of only `InvoiceDate`
* Ensures no data loss from delayed events

---

### **Late-Arriving Data**

* Handled via:

  * **Upsert strategy** in Clean Layer
  * Primary key: `InvoiceNo + StockCode`
* Prevents duplication and ensures consistency

---

### **Streaming + Batch Unification**

* Streaming data processed in **micro-batches**
* Ensures consistency with batch processing logic

---

### **Feature Refresh Strategy**

* Daily recomputation using:

  * **Sliding window (7, 30, 90 days)**
* Only recompute for:

  * Customers with recent activity
* Avoids full recomputation cost