# Hospital PPE Consumption & Inventory Replenishment Analytics (April 2026)

## 1. Introduction

In high-acuity healthcare environments, personal protective equipment (PPE) supply chain continuity directly impacts patient safety, staff health, and regulatory compliance. During periods of fluctuating demand, hospitals face dual challenges: avoiding stock-outs in critical care departments (such as ICUs and Emergency Departments) while controlling procurement expenditures and preventing over-ordering.

This project delivers an end-to-end Power BI business intelligence solution designed to monitor April 2026 PPE utilization, detect abnormal shift-level consumption surges, evaluate on-hand asset values across hospital storage locations, and identify inventory compensation gaps against established reorder thresholds.

The resulting analytical solution is divided into two distinct executive reporting pages:
1. **PPE Consumption & Utilisation Analysis Report Apr-2026**: Focuses on operational consumption velocity, departmental allocation, peak burn days, and rolling demand smoothing.
2. **Financial Valuation & Inventory Replenishment Report**: Focuses on asset valuation, procurement pipeline tracking, storage unit distribution, and inventory deficit reconciliation.

---

## 2. Data Source

* **Generation Method**: Synthetically engineered using **Python** (Pandas, NumPy, and Datetime libraries).
* **Simulation Characteristics**:
  * Realistic clinical distribution patterns reflecting higher utilization across critical care departments (Emergency, Intensive Care Unit, Operating Theatres) compared to outpatient and general medical units.
  * Shift-based variations and weekend smoothing across a 30-day temporal baseline (April 1 to April 30, 2026).
  * Multi-echelon logistics incorporating central warehouse storage alongside point-of-use clinical units.
  * Stochastic inbound replenishment orders with supplier lead times and expected arrival dates.

---

## 3. Repository Structure

```text
├── data/
│   ├── Consumption_Log_April.csv       # Daily PPE transactional usage log
│   ├── Daily_Inventory_April.csv       # Daily multi-location stock on-hand snapshots
│   ├── Inbound_Orders_April.csv        # Procurement orders, delivery statuses, and dates
│   └── Item_Master_April.csv           # SKU catalog, categories, unit costs, and reorder points
├── PPE_Analysis_April_2026.pbix        # Interactive Power BI report and semantic model
└── README.md                           # Project technical documentation
```

---

## 4. Data Description

The data architecture integrates four primary operational tables alongside an engineered calendar dimension table:

### 4.1. `Item_Master_April.csv` (Dimension Table)
Defines the SKU catalog, item classifications, procurement cost baseline, and safety thresholds.
* **`ItemID`** *(Text / Key)*: Unique identifier for each PPE SKU (e.g., `GLV-NIT-M`, `MSK-N95-01`).
* **`ItemName`** *(Text)*: Full commercial or clinical product name.
* **`Category`** *(Text)*: PPE group (e.g., Gloves, Masks & Respirators, Gowns, Face Shields).
* **`UnitCost`** *(Decimal Number)*: Contracted acquisition price per unit ($ USD).
* **`ReorderPoint`** *(Whole Number)*: Minimum safety stock inventory threshold trigger.

### 4.2. `Consumption_Log_April.csv` (Fact Table)
Granular record of physical PPE consumption across clinical units.
* **`TransactionID`** *(Whole Number / Key)*: Unique identifier for the consumption log record.
* **`TransactionDate`** *(Date)*: Date of consumption (`YYYY-MM-DD`).
* **`ItemID`** *(Text / Foreign Key)*: Foreign key referencing `Item_Master_April[ItemID]`.
* **`Department`** *(Text)*: Consuming hospital department (e.g., ICU, Emergency, Surgery, General Ward).
* **`QtyConsumed`** *(Whole Number)*: Physical count of units used.

### 4.3. `Daily_Inventory_April.csv` (Periodic Snapshot Fact Table)
End-of-day physical stock count by item and location.
* **`Date`** *(Date)*: Snapshot record date (`YYYY-MM-DD`).
* **`ItemID`** *(Text / Foreign Key)*: Foreign key referencing `Item_Master_April[ItemID]`.
* **`Location`** *(Text)*: Physical inventory location (e.g., Central Warehouse, ER Supply, ICU Shared Supply).
* **`QtyOnHand`** *(Whole Number)*: Physical units stored at the specified location at close of day.

### 4.4. `Inbound_Orders_April.csv` (Fact Table)
Purchase order log for inbound supplier replenishment.
* **`OrderID`** *(Text / Key)*: Unique purchase order identifier.
* **`OrderDate`** *(Date)*: Date the replenishment order was authorized.
* **`ItemID`** *(Text / Foreign Key)*: Foreign key referencing `Item_Master_April[ItemID]`.
* **`QtyOrdered`** *(Whole Number)*: Volume of units purchased.
* **`ExpectedArrival`** *(Date)*: Estimated supplier delivery date.
* **`Status`** *(Text)*: Order tracking status (e.g., Shipped, In-Transit, Delivered).

### 4.5. `Date` Table (Generated Calendar Dimension)
Generated continuous calendar dimension spanning the entire project timeframe.
* **`Date`** *(Date)*: Primary key (`2026-04-01` to `2026-04-30`).
* **`Day Month`** *(Text)*: Short-form date string (`01 Apr`, `02 Apr`) used for trend visualization axes, sorted chronologically by calendar date.
* **`DayOfWeek`**, **`Month`**, **`Year`**: Standard calendar attributes.

---

## 5. Model Structure & Relationships

The Power BI data model utilizes a Star Schema extended to handle multiple fact tables referencing shared dimension tables.

```
       +-----------------------+              +-----------------------+
       |         Date          |              |   Item_Master_April   |
       |-----------------------|              |-----------------------|
       | [Date] (PK)           |              | [ItemID] (PK)         |
       | [Day Month]           |              | [ItemName]            |
       | ...                   |              | [Category]            |
       +-----------+-----------+              | [UnitCost]            |
                   |                          | [ReorderPoint]        |
                   | 1                        +-----------+-----------+
                   |                                      | 1
         +---------+---------+                            |
         |                   |                            |
       * |                 * |                            |
+--------+--------+ +--------+--------+                   |
| Consumption_Log | | Daily_Inventory |                   |
|-----------------| |-----------------|                   |
| TransactionDate | | Date            |                   |
| ItemID      *---|---+ ItemID        |<------------------+ *
| Department      | | Location        |                   |
| QtyConsumed     | | QtyOnHand       |                   |
+-----------------+ +-----------------+                   |
                             *                            |
                             | 1 (Active: OrderDate)      |
                    +--------+--------+                   |
                    | Inbound_Orders  |                   |
                    |-----------------|                   |
                    | OrderDate       |                   |
                    | ExpectedArrival | (Inactive)        |
                    | ItemID      *---+-------------------+
                    | QtyOrdered      |
                    +-----------------+
```

### Relationship Details:
1. **`Date[Date]` to `Consumption_Log_April[TransactionDate]`**:
   * Cardinality: One-to-Many (`1:*`), Active.
   * Cross-filter Direction: Single (`Date` filters `Consumption_Log_April`).
2. **`Date[Date]` to `Daily_Inventory_April[Date]`**:
   * Cardinality: One-to-Many (`1:*`), Active.
   * Cross-filter Direction: Single (`Date` filters `Daily_Inventory_April`).
3. **`Date[Date]` to `Inbound_Orders_April[OrderDate]`**:
   * Cardinality: One-to-Many (`1:*`), Active.
   * Cross-filter Direction: Single (`Date` filters `Inbound_Orders_April`).
4. **`Date[Date]` to `Inbound_Orders_April[ExpectedArrival]`**:
   * Cardinality: One-to-Many (`1:*`), Inactive (activated via analytical expressions when evaluating arrival lead time).
5. **`Item_Master_April[ItemID]` to Fact Tables**:
   * One-to-Many (`1:*`) active relationships connecting the item dimension to `Consumption_Log_April[ItemID]`, `Daily_Inventory_April[ItemID]`, and `Inbound_Orders_April[ItemID]`.

---

## 6. Power BI Reports Overview

### Report 1: "PPE Consumption & Utilisation Analysis Report Apr-2026"

This report provides operations managers and clinical nurse supervisors with a clear breakdown of physical PPE usage, burn velocity, and department-level consumption behaviors.

```text
+------------------------------------------------------------------------------------+
|  [Date Range Slicer]   |   [Category Slicer]   |   [Department Multi-Select]       |
+------------------------------------------------------------------------------------+
| [ Total Consumed ]  [ Daily Burn Rate ]  [ Peak Consumption ]  [ Stock on Hand ]   |
|     128,450 units       4,282 units/day       6,810 units           42,100 units   |
+-------------------------------------------------+----------------------------------+
| Clustered Bar Chart:                            | Matrix:                          |
| Departmental Volume by Category                 | Department x Item Breakdown      |
| Y-Axis: Department, Category                    | Rows: Department                 |
| X-Axis: Total Units Consumed                    | Columns: ItemID                  |
| Tooltips: Department Share %, Daily Burn Rate   | Values: Units Consumed,          |
+-------------------------------------------------+         Department Share %,      |
| Line Chart: Daily Consumption & 7-Day Trend     |         Daily Burn Rate          |
| X-Axis: Day Month                               | Formatting: Data Bars            |
| Y-Axis: Total Units Consumed, Consumption 7D MA |                                  |
| Tooltips: DoD Consumption Growth %              |                                  |
+-------------------------------------------------+----------------------------------+
```

* **KPI Summary Cards**:
  * **Total Units Consumed**: Total physical count of PPE utilized within the filtered date and department context.
  * **Daily Burn Rate**: Average daily consumption velocity across active transaction days.
  * **Peak Daily Consumption**: Maximum single-day unit depletion volume, displaying peak date context.
  * **Stock on Hand**: Latest snapshot balance of physical units remaining in facility inventory.
* **Visual 1: Clustered Bar Chart (Departmental Usage by Category)**:
  * **Y-Axis**: Hierarchical breakdown by `Department` and `Category`.
  * **X-Axis**: `Total Units Consumed`.
  * **Tooltips**: `Department Consumption Share` (% of total consumption generated by that unit) and `Daily Burn Rate`.
* **Visual 2: Line Chart (Consumption Velocity & Moving Average)**:
  * **X-Axis**: `Day Month` (chronologically sorted by calendar date).
  * **Y-Axis**: `Total Units Consumed` (actual daily usage) and `Consumption 7D MA` (7-day smoothed moving average line).
  * **Tooltips**: `DoD Consumption Growth` (day-over-day percentage change identifying isolated spikes).
* **Visual 3: Matrix (Detailed Departmental & Item Consumption)**:
  * **Rows**: `Department`.
  * **Columns**: `ItemID`.
  * **Values**: `Units Consumed`, `Department Share`, and `Daily Burn Rate`.
  * **Formatting**: In-cell **Data Bars** applied across volume metrics for rapid visual ranking.

---

### Report 2: "Financial Valuation & Inventory Replenishment Report"

This report equips hospital procurement officers and supply chain directors to assess asset distribution, reconcile inbound supplier orders against consumption costs, and identify inventory shortfalls relative to safety points.

```text
+------------------------------------------------------------------------------------+
|  [Date Period Slicer]     |     [Location Filter]     |     [Category Filter]      |
+------------------------------------------------------------------------------------+
| [ Total Consumed Cost ]  [ Daily Consumed ]  [ Inventory Value ]  [ Pipeline Spend ]
|       $94,220.00            $3,140.67/day         $63,400.00          $31,250.00   |
+-------------------------------------------------+----------------------------------+
| Clustered Column Chart:                         | Pie Chart:                       |
| Financial Comparison by Category                | Inventory Valuation by Location  |
| X-Axis: Category                                | Legend: Location                 |
| Values: Total Consumed Cost,                    | Values: Current Inventory Value  |
|         Inbound Pipeline Value                  | Tooltips: Stock on Hand          |
+-------------------------------------------------+----------------------------------+
| Line Chart: Valuation vs. Safety Par Baseline   | Table: Inventory Compensation    |
| X-Axis: Date                                    | Columns: ItemID, Category,       |
| Y-Axis: Daily Inventory Value,                  |          UnitCost, Stock on Hand,|
|         Target Reorder Stock Value              |          Inbound Pipeline Qty,   |
| Tooltips: Consumed Cost, Pipeline Value, Stock  |          Net Available Stock,    |
| Reference: Safety Buffer Line ($59,850 baseline)|          ReorderPoint,           |
|                                                 |          Stock Status (Color Tag)|
+-------------------------------------------------+----------------------------------+
```

* **Global Interactive Slicers**:
  * `Date`: Date slider controlling the reporting window.
  * `Location`: Multi-select dropdown filtering inventory storage units.
  * `Category`: Product category dropdown.
* **KPI Summary Cards**:
  * **Total Consumed Cost**: Financial valuation ($) of all depleted inventory.
  * **Daily Consumed**: Average financial burn rate per day ($/Day).
  * **Current Inventory Value**: Total asset value ($) of stock on hand evaluated at the latest date snapshot.
  * **Inbound Pipeline Value**: Total dollar commitment for incoming purchase orders.
* **Visual 1: Clustered Column Chart (Category Financial Comparison)**:
  * **X-Axis**: `Category`.
  * **Y-Axis**: `Total Consumed Cost` and `Inbound Pipeline Value`.
  * **Purpose**: Compares financial depletion against procurement commitments across categories without cluttering the axis with baseline reorder thresholds.
* **Visual 2: Line Chart (Asset Valuation vs. Target Reorder Level)**:
  * **X-Axis**: `'Date'[Date]`.
  * **Y-Axis**: `Daily Inventory Value` alongside the constant baseline `Target Reorder Stock Value` ($59,850 safety line).
  * **Tooltips**: Daily consumed cost, inbound pipeline spend, and physical units on hand.
* **Visual 3: Pie Chart (On-Hand Valuation by Storage Location)**:
  * **Legend**: `Location` (e.g., Central Warehouse, ER Supply, ICU Shared Supply).
  * **Values**: `Current Inventory Value`.
  * **Tooltips**: `Stock on Hand` (physical unit count).
  * **Purpose**: Identifies capital distribution to evaluate whether internal inventory transfers can resolve departmental deficits before placing supplier orders.
* **Visual 4: Table (Inventory Compensation & Reorder Status Grid)**:
  * **Columns**:
    1. `Item_Master_April[ItemID]`
    2. `Item_Master_April[Category]`
    3. `Item_Master_April[UnitCost]`
    4. `Current Stock On Hand`
    5. `Inbound Pipeline Qty`
    6. `Net Available Stock`
    7. `Item_Master_April[ReorderPoint]`
    8. `Stock Status`
  * **Formatting**: Dynamic conditional formatting applied as a background color to the `Stock Status` column (Red for below reorder threshold, Amber for low buffer, Green for sufficient stock).

---

## 7. Setup & Reproduction Guide

Follow these steps to recreate or update the Power BI project:

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/your-org/ppe-consumption-inventory-powerbi.git
   cd ppe-consumption-inventory-powerbi
   ```

2. **Verify Data Directory**:
   Ensure all four CSV files are stored in the `/data` folder:
   * `Consumption_Log_April.csv`
   * `Daily_Inventory_April.csv`
   * `Inbound_Orders_April.csv`
   * `Item_Master_April.csv`

3. **Open Power BI Desktop**:
   * Open `PPE_Analysis_April_2026.pbix`.
   * If re-linking data sources, go to **Home > Transform Data > Data Source Settings**, select the folder path matching your local directory, and click **Apply Changes**.

4. **Verify Date Dimension**:
   * Ensure the `'Date'` table is marked as an official Date Table:
     * Right-click `'Date'` in the Fields pane > **Mark as date table** > Select `Date` column.

5. **Verify Relationships**:
   * Switch to the **Model View** and verify that all 1-to-many relationships match the schema outlined in Section 5. Ensure `Cross-filter direction` is set to **Single**.

---

## 8. Power BI Configuration & Best Practices

* **Measure Cleanliness Rule**:
  In accordance with governance standards, only measures directly referenced by the active cards, charts, matrix visual, and table grids are retained in the dedicated `_DAX Measures` table. Auxiliary or intermediate calculations are consolidated directly within dependent expressions to optimize data model performance and prevent clutter.
* **Data Type Enforcement**:
  * Unit cost, consumed spend, pipeline spend, and inventory valuation columns are formatted as **Currency (`$#,##0.00`)**.
  * Quantities and counts (`QtyConsumed`, `QtyOnHand`, `QtyOrdered`, `ReorderPoint`) are configured as **Whole Number (`#,##0`)**.
  * Percentages (`Department Consumption Share`, `DoD Consumption Growth`) are explicitly formatted as **Percentage (`0.0%`)**.
* **Date Sorting Configuration**:
  * The `'Date'[Day Month]` column has its **Sort by Column** property set to `'Date'[Date]` to prevent alphabetical sorting errors across chart axes.
* **Snapshot Aggregation Handling**:
  * Measures accessing `Daily_Inventory_April` use semi-additive aggregation logic (`LASTDATE`) to ensure snapshot figures represent closing balances without incorrectly summing across multi-day selections.
