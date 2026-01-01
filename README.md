# 🚲 Adventure Works: Global Sales & Returns Dashboard

![Power Bi](https://img.shields.io/badge/power_bi-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-00758F?style=for-the-badge&logo=analythical&logoColor=white)
![M Language](https://img.shields.io/badge/M_Language-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![CSV](https://img.shields.io/badge/csv-20232A?style=for-the-badge&logo=csv&logoColor=white)

## 📖 Project Overview
This Power BI project provides a detailed analysis of **Adventure Works**, a global manufacturing company selling cycling equipment across three major continents (North America, Europe, Australia). The report focuses on bridging the gap between high-volume sales and profitability while monitoring return rates to ensure operational efficiency.

The dashboard integrates raw CSV data into a robust **Star Schema** model, utilizing advanced DAX for time intelligence, dynamic target setting, and "What-If" scenario planning.

---

## 🏗️ Data Structure & Modeling

### 🔌 Tech Stack & Connections
* **Power BI Desktop:** The primary visualization and modeling tool.
* [cite_start]**Power Query (M Language):** Used for ETL (Extract, Transform, Load) and to generate the schema-only Measure table.
* **Source Data:** Raw CSV files (`Sales Data`, `Returns Data`, `Customer Lookup`, `Product Lookup`, `Calendar Lookup`, `Product Categories Lookup`).

### 📊 Relationship Diagram (Star Schema)
The data model uses a Star Schema architecture. Dimension (Lookup) tables filter the Fact (Data) tables.

```mermaid
erDiagram
    CALENDAR_LOOKUP ||--o{ SALES_DATA : "Filters by Date"
    CALENDAR_LOOKUP ||--o{ RETURNS_DATA : "Filters by Date"
    PRODUCT_LOOKUP ||--o{ SALES_DATA : "Filters by ProductKey"
    PRODUCT_LOOKUP ||--o{ RETURNS_DATA : "Filters by ProductKey"
    CUSTOMER_LOOKUP ||--o{ SALES_DATA : "Filters by CustomerKey"
    PRODUCT_CATEGORIES_LOOKUP ||--o{ PRODUCT_LOOKUP : "Categorizes Products"
    PRICE_ADJUSTMENT ||--|{ MEASURE_TABLE : "Disconnected Table for What-If"
```

## 🧮 DAX Measures: Detailed Explanation
This project uses a dedicated Measure Table created via M-Code. Below is the breakdown of the logic used, categorized by function.

### 1. 💰 Core Financials (Iterators vs. Aggregators)
**The Challenge:** Revenue and Cost data do not exist in a single column in the source files. The Sales Data table has quantity, but prices live in Product Lookup.
**The Solution:** We use SUMX iterators to calculate row-by-row before summing.

#### Total Cost
* **Formula:** `SUMX('Sales Data', 'Sales Data'[OrderQuantity] * RELATED('Product Lookup'[ProductCost]))`
* **Why/How:** SUMX iterates through every row of the Sales table. RELATED fetches the cost from the Product table for that specific row.

#### Total Revenue
* **Formula:** `SUMX('Sales Data', 'Sales Data'[OrderQuantity] * RELATED('Product Lookup'[Discount Price]))`
* **Why:** Ensures discounts and specific product prices are applied accurately at the transaction level.

#### Total Profit
* **Formula:** `[Total Revenue] - [Total Cost]`

### 2. ↩️ Returns Analysis & Category Isolation
**The Challenge:** We need to see global returns, but also specifically isolate "Bikes" as they are high-value items.
**The Solution:** CALCULATE is used to override filter context.

#### Return Rate
* **Formula:** `DIVIDE([Quantity Returned], [Quantity Sold], "No Sales")`
* **Why:** DIVIDE is safer than the / operator because it handles division-by-zero errors automatically.

#### Bike Returns
* **Formula:** `CALCULATE([Total Returns], 'Product Categories Lookup'[CategoryName] = "Bikes")`
* **How:** This measure ignores other filters on category and forces the calculation to look only at "Bikes".

#### % of All Returns
* **Formula:** `DIVIDE([Total Returns], [All Returns])` where `[All Returns]` uses `CALCULATE(..., ALL('Returns Data'))`.
* **Why:** The ALL function removes all filters from the returns table, providing a grand total denominator to calculate the percentage share.

### 3. 📅 Time Intelligence (Trends & Rolling Averages)
**The Challenge:** Sales are volatile. We need to compare performance against the previous month and smooth out daily noise.
**The Solution:** DAX Time Intelligence functions.

#### Previous Month Metrics
* **Formula:** `CALCULATE([Total Revenue], DATEADD('Calendar Lookup'[Date], -1, MONTH))`
* **Why:** DATEADD shifts the date context back exactly one month, allowing for MoM (Month-over-Month) comparison columns.

#### 90-Day Rolling Profit
* **Formula:** `CALCULATE([Total Profit], DATESINPERIOD('Calendar Lookup'[Date], MAX('Calendar Lookup'[Date]), -90, DAY))`
* **How:** DATESINPERIOD creates a dynamic window of the last 90 days from the current date in the visual context, smoothing out short-term fluctuations.

#### 10-Day Rolling Revenue
* **Formula:** Uses `DATESINPERIOD` with `-10, DAY`.

### 4. 🎯 Dynamic Targets & Gaps
**The Challenge:** Static goals become obsolete quickly.
**The Solution:** Programmatic targets based on recent performance.

**Logic:** The target is set to 110% (1.1 multiplier) of the previous month's performance.
* **Revenue Target:** `[Previous Month Revenue] * 1.1`
* **Order Target:** `[Previous Month Orders] * 1.1`

#### Target Gaps
* **Formula:** `[Total Revenue] - [Revenue Target]`
* **Usage:** Used in KPI cards to show "Distance to Goal" with conditional formatting (Red/Green).

### 5. 👤 Customer Detail Logic
**The Challenge:** We want to show customer names in a list, but prevent the measure from calculating a "Total" name at the bottom of a table.
**The Solution:** HASONEVALUE.

#### Full Name (Customer Detail)
* **Formula:** `IF(HASONEVALUE('Customer Lookup'[CustomerKey]), MAX('Customer Lookup'[Full Name]), "Multiple Customers")`
* **Why:** If the visual row represents one customer, show their name. If it's the "Total" row (which represents many customers), display "Multiple Customers" or a blank dash.

### 6. 🔮 What-If Scenario (Price Adjustment)
**The Challenge:** Stakeholders want to simulate: "What happens to profit if we raise prices by 5%?"
**The Solution:** A disconnected parameter table.

* **Adjusted Price:** `[Average Retail Price] * (1 + 'Price Adjustment'[Price Adjustment Value])`
* **Adjusted Revenue:** Re-runs the SUMX logic using `[Adjusted Price]` instead of the standard price.
* **Impact:** This allows users to use a slicer to dynamically update financial projections without altering the underlying data.

## 🛠️ Technical Implementation Details
### The M-Table Hack
Instead of a standard calculated table, the Measure Table was generated using M-Language binary decompression:
```powerquery
Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i44FAA==", ...
```
**Why?** This creates a table with no data source dependencies, strictly for housing measures, keeping the field list clean.

### Formatting
* **Currency:** All monetary values are formatted with en-US culture hints ($#,0.00).
* **Handling Nulls:** Measures like Total Orders (Customer Detail) return "-" if multiple values exist, ensuring clean visual tables.

## 🚀 How to Run
1. **Download:** Ensure all .csv files and the .pbix are in the same folder structure.
2. **Connections:** Open Power BI > Transform Data > Data Source Settings > Change Source to your local directory.
3. **Refresh:** Click "Refresh" to load the latest data from the CSVs.