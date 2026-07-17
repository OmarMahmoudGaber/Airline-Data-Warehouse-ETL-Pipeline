<<<<<<< HEAD
# Airline-Data-Warehouse-ETL-Pipeline
=======
# ✈️ Airline Enterprise Data Warehouse & IBM DataStage ETL Pipeline

## 📖 Project Overview
This project delivers a robust, end-to-end Enterprise Data Warehouse (EDW) and automated ETL pipeline for a major airline company. Designed to support executive decision-making, the analytical environment consolidates complex business processes across flight activity, frequent flyer engagement, reservation finances, and customer care interactions. 

The architecture bridges the gap between raw operational data and actionable business intelligence through rigorous dimensional modeling, optimized physical design, and resilient ETL orchestration.

---

## 🏗️ Data Architecture & Logical Modeling

The Data Warehouse utilizes a **Galaxy Schema (Fact Constellation)** approach. This architecture deploys multiple fact tables sharing conformed dimensions to accommodate the differing grains of the airline's business processes without creating sparse or overly complex single tables.

### Entity Relationship Diagram (ERD)
![ERD Model](Data%20Modeling/Airline%20ERD.png)

### Dimensional Model (Galaxy Schema)
![Dimensional Model](Data%20Modeling/Airline%20Dimensional%20Modeling.png)

### Dimensional Models
*   **`DIM_FLIGHT`:** A flattened, denormalized model. The ETL pipeline constructs this by joining original normalized OLTP data (Routes, Origin Airports, Destination Airports, Aircraft) into a single dimension to eliminate complex joins during downstream querying.
*   **`DIM_DATE` (Role-Playing Dimension):** Generated programmatically to handle all time-based reporting (Booking Date, Flight Date, Interaction Date). Includes calendar hierarchies (Year, Quarter, Month, Day) and weekend flags.
*   **`DIM_PASSENGER` (SCD Type 2):** Tracks the lifecycle of a customer. Because frequent flyers upgrade or downgrade their tiers, this table utilizes `EFFECTIVE_DATE`, `EXPIRATION_DATE`, and `IS_CURRENT` flags to preserve historical accuracy regarding their status at the time of a specific flight.
*   **Standard Dimensions:** `DIM_FARE_CLASS`, `DIM_CHANNEL`, `DIM_PROMOTION`, and `DIM_INTERACTION` act as highly optimized lookup tables for static descriptive attributes.

### Transaction & Snapshot Fact Models
*   **`FACT_FLIGHT_ACTIVITY` (Transaction Fact):** Grain is set at one row per passenger per completed flight segment. Consolidates flight activity, miles accrued/redeemed, upgrades, and promotional responses.
*   **`FACT_RESERVATION_FINANCES` (Transaction Fact):** Grain is set at one row per ticket issued. Centralizes revenue and cost metrics (`FINAL_TICKET_AMOUNT_USD`, `EST_FLIGHT_COST_ALLOCATION`, `PROFIT_MARGIN_USD`).
*   **`FACT_CUSTOMER_CARE` (Accumulating Snapshot Fact):** Grain is set at one row per customer interaction. Tracks the lifecycle of inquiries and complaints from initiation to resolution, capturing resolution time and CSAT scores.

---

## ⚙️ ETL Pipeline Implementation (IBM DataStage)

The extraction, transformation, and loading logic is engineered for high throughput and recoverability using IBM DataStage.

### 1. Orchestration & Dependency Management
Pipeline dependencies are strictly enforced via a Master Sequence Job to ensure data integrity and failure recoverability.

![Master Sequence Job](Jobs_Screenshots/Sequence%20Job/Sequence%20Diagram.png)

1.  **Phase 1 (`Dimensions_Sequencer`):** Standard dimensions run concurrently.
2.  **Phase 2 (`Dim_Passengers`):** Runs sequentially after the standard dimensions to handle complex Slowly Changing Dimension (SCD Type 2) logic safely.
3.  **Phase 3 (`Facts_Sequencer`):** Triggers Fact tables only after all surrogate keys are available in the dimension tables.
4.  **Phase 4 (`Marts_sequencer`):** Finalizes the run by triggering the downstream departmental aggregations.

### 2. Dimension Processing
Dimensions are populated using dedicated jobs that extract source data, apply transformations, generate Surrogate Keys (SKs), and load the target Oracle tables.

**Date Dimension Generation:**
Utilizes a Row Generator and Transformer to sequentially build time attributes programmatically, removing the need for external flat files.
![Dim Date Job](Jobs_Screenshots/Dimensions%20Jobs/Dim_Date%20Job.png)

**Flight Dimension Denormalization:**
Performs sequential lookups across five distinct source systems to flatten the operational routing and aircraft data into a single, highly performant analytical dimension.
![Dim Flights Job](Jobs_Screenshots/Dimensions%20Jobs/Dim_Flights%20Job.png)

### 3. Fact Processing
Fact table pipelines are designed for heavy analytical loads. The core transactional streams are joined with necessary operational artifacts before passing through a centralized lookup stage. Surrogate keys are resolved via parallel lookups against the previously loaded dimensions before final transformations and target loading.

**Flight Activity Fact:**
![Fact Flight Activity Job](Jobs_Screenshots/Facts%20Jobs/Fact_Flight_Activity%20Job.png)

**Reservation Finances Fact:**
![Fact Reservation Finance Job](Jobs_Screenshots/Facts%20Jobs/Fact_Reservation_Finance%20Job.png)

**Customer Care Fact:**
![Fact Customer Care Job](Jobs_Screenshots/Facts%20Jobs/Fact_Customer_Care%20Job.png)

---

## 📊 Data Marts & Aggregations

To provide immediate, flexible dashboarding for specific departments, automated Data Mart pipelines calculate required KPIs:

*   **`MART_FREQUENT_FLYER_BEHAVIOR` (Marketing Mart):** Aggregates granular flight activity with the Date Dimension via specialized join and aggregation stages to track overall loyalty and promotional engagement.
    ![Marketing Mart Job](Jobs_Screenshots/Marts%20Jobs/Marketing%20Mart%20Job.png)
*   **`MART_CHANNEL_PROFITABILITY`:** Aggregates finance data at the Channel and Month level to track revenue and profit margins.

---

## 🚀 Performance Optimization Strategies

To ensure optimal query execution plans against massive airline datasets, specific physical design optimizations were deployed:
*   **Range Partitioning:** Applied to all fact tables based on `DATE_KEY`. This guarantees partition pruning for timeframe-specific executive queries (e.g., "Q3 2026") and streamlines historical data archiving.
*   **Bitmap Indexes:** Applied to low-cardinality foreign keys (`FARE_CLASS_SK`, `CHANNEL_SK`, `PROMOTION_SK`) to allow rapid filtering and aggregation of static categories.
*   **B-Tree Indexes:** Applied to high-cardinality keys (`PASSENGER_SK`, `FLIGHT_SK`) to provide optimal lookup efficiency for millions of unique identifiers.

---

## 💡 Executive Business Intelligence (SQL Analytics)

The following analytical queries demonstrate the physical model's capability to answer complex executive questions. 
### Q1: Frequent Flyer Tier Distribution
*Calculates what proportion of frequent flyers have Gold, Platinum, or Titanium status.*

```sql
SELECT
    TIER_NAME,
    COUNT(PASSENGER_SK) AS PASSENGER_COUNT,
    ROUND(COUNT(PASSENGER_SK) * 100.0 / SUM(COUNT(PASSENGER_SK)) OVER(), 2) AS PROPORTION_PCT
FROM DIM_PASSENGER
WHERE IS_CURRENT = 'Y' 
  AND TIER_NAME IN ('Gold', 'Platinum', 'Titanium')
GROUP BY TIER_NAME;
```

### Q2: Upgrade Frequencies by Tier
*Analyzes how often frequent flyers upgrade.*

```sql
SELECT
    p.TIER_NAME,
    COUNT(f.FLIGHT_ACTIVITY_ID) AS TOTAL_FLIGHTS,
    SUM(f.IS_UPGRADED) AS TOTAL_UPGRADES,
    ROUND(SUM(f.IS_UPGRADED) * 100.0 / COUNT(f.FLIGHT_ACTIVITY_ID), 2) AS UPGRADE_RATE_PCT
FROM FACT_FLIGHT_ACTIVITY f
JOIN DIM_PASSENGER p ON f.PASSENGER_SK = p.PASSENGER_SK
WHERE p.TIER_NAME != 'None' 
GROUP BY p.TIER_NAME;
```

### Q3: Promotional Campaign Response Rates
*Measures whether frequent flyers respond to special fare promotions.*

```sql
SELECT
    p.TIER_NAME,
    pr.PROMO_NAME,
    COUNT(f.FLIGHT_ACTIVITY_ID) AS TOTAL_BOOKINGS,
    SUM(f.RESPONDED_TO_PROMO) AS PROMO_RESPONSES,
    ROUND(SUM(f.RESPONDED_TO_PROMO) * 100.0 / COUNT(f.FLIGHT_ACTIVITY_ID), 2) AS RESPONSE_RATE_PCT
FROM FACT_FLIGHT_ACTIVITY f
JOIN DIM_PASSENGER p ON f.PASSENGER_SK = p.PASSENGER_SK
JOIN DIM_PROMOTION pr ON f.PROMOTION_SK = pr.PROMOTION_SK
WHERE pr.PROMO_ID != -1 
GROUP BY p.TIER_NAME, pr.PROMO_NAME;
```

### Q4: Overnight Stay Analysis
*Determines how long their overnight stays are based on passenger tier.*

```sql
SELECT
    p.TIER_NAME,
    AVG(f.OVERNIGHT_STAY_NIGHTS) AS AVG_STAY_DURATION_NIGHTS,
    MAX(f.OVERNIGHT_STAY_NIGHTS) AS MAX_STAY_DURATION_NIGHTS
FROM FACT_FLIGHT_ACTIVITY f
JOIN DIM_PASSENGER p ON f.PASSENGER_SK = p.PASSENGER_SK
WHERE f.OVERNIGHT_STAY_NIGHTS > 0
GROUP BY p.TIER_NAME;
```

### Q5: Multi-Channel Profitability
*Analyzes company profit through multiple reservation channels.*

```sql
SELECT
    c.CHANNEL_NAME,
    c.CHANNEL_TYPE,
    SUM(f.FINAL_TICKET_AMOUNT_USD) AS TOTAL_REVENUE,
    SUM(f.EST_FLIGHT_COST_ALLOCATION) AS TOTAL_COST,
    SUM(f.PROFIT_MARGIN_USD) AS TOTAL_PROFIT
FROM FACT_RESERVATION_FINANCES f
JOIN DIM_CHANNEL c ON f.CHANNEL_SK = c.CHANNEL_SK
GROUP BY c.CHANNEL_NAME, c.CHANNEL_TYPE
ORDER BY TOTAL_PROFIT DESC;
```

### Q6: Customer Care Resolution Efficiency
*Analyzes customer care interaction type and problem severity.*

```sql
SELECT
    i.INTERACTION_TYPE,
    i.SEVERITY,
    COUNT(c.CUSTOMER_CARE_ID) AS TOTAL_ISSUES,
    AVG(c.RESOLUTION_TIME_DAYS) AS AVG_RESOLUTION_DAYS,
    AVG(c.CSAT_SCORE) AS AVG_CUSTOMER_SATISFACTION
FROM FACT_CUSTOMER_CARE c
JOIN DIM_INTERACTION i ON c.INTERACTION_SK = i.INTERACTION_SK
GROUP BY i.INTERACTION_TYPE, i.SEVERITY
ORDER BY TOTAL_ISSUES DESC;
```
>>>>>>> 881b500cd2d7d3fe4508bef8a1f8b880e3b1f0c7
