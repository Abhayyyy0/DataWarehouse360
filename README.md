# DataWarehouse360 (SQL Server)

A practical, end-to-end SQL Server data warehouse showcasing Medallion Architecture (Bronze/Silver/Gold), ETL pipelines, dimensional modeling, and analytics. Inspired by `Abhayyyy0/SQL-data-warehouse`.

### Key Features
- **Medallion Layers**: Raw ingestion (Bronze), cleaned/standardized (Silver), star schema for BI (Gold)
- **ETL Pipelines**: Repeatable SQL scripts for loading, transforming, and modeling
- **Dimensional Model**: Fact and dimension tables optimized for analytics
- **Analytics**: Ready-to-run SQL queries for common business questions

## Prerequisites
- SQL Server 2019+ (Developer/Express) or Azure SQL Database
- SSMS or Azure Data Studio
- CSV files for source data (ERP/CRM). If using the reference repository, see its `datasets/` directory

## Getting Started
1. Create a new SQL Server database, e.g. `DW_Sales`
2. Create schemas for each layer: bronze , silver , gold 
3. Access the data through 'datasets' folder in the repositary
   ```sqlRD


## Medallion Architecture
- **Bronze**: Raw tables as-is from sources; minimal typing; append-only loads
- **Silver**: Cleaned data with standardized types, deduplicated entities, conformed dimensions
- **Gold**: Business-ready model using a star schema (facts + dimensions) for BI

## Dimensional Model (example)
- Dimensions: `gold.DimDate`, `gold.DimCustomer`, `gold.DimProduct`, `gold.DimSalesRep`
- Fact: `gold.FactSales`

Surrogate keys are generated in dimensions; facts reference them via foreign keys.

## Typical ETL Order
1. Bronze: create tables and load CSVs
2. Silver: standardize datatypes, clean text, resolve duplicates, integrate sources
3. Gold: build dimensions, assign surrogate keys, build fact tables, add indexes

Example: Build `DimCustomer` and join CRM+ERP customers with survivorship rules, then map to `FactSales` via surrogate key.

## Example SQL Snippets
- Create Date Dimension (partial):
  ```sql
  CREATE TABLE gold.DimDate (
    DateKey INT PRIMARY KEY,
    [Date] DATE NOT NULL,
    Year INT, Quarter INT, Month INT, Day INT,
    MonthName NVARCHAR(20), DayName NVARCHAR(20)
  );
  ```
- Generate surrogate keys and upsert pattern:
  ```sql
  MERGE gold.DimCustomer AS tgt
  USING (SELECT DISTINCT ... FROM silver.Customers) AS src
  ON (tgt.BusinessCustomerId = src.BusinessCustomerId)
  WHEN NOT MATCHED THEN
    INSERT (BusinessCustomerId, CustomerName, City, Country)
    VALUES (src.BusinessCustomerId, src.CustomerName, src.City, src.Country)
  WHEN MATCHED THEN
    UPDATE SET CustomerName = src.CustomerName, City = src.City, Country = src.Country;
  ```
- Fact load (joining conformed dimensions):
  ```sql
  INSERT INTO gold.FactSales (DateKey, CustomerKey, ProductKey, SalesRepKey, Quantity, Amount)
  SELECT d.DateKey, dc.CustomerKey, dp.ProductKey, dr.SalesRepKey, s.Quantity, s.Amount
  FROM silver.Sales s
  JOIN gold.DimDate d ON d.[Date] = s.OrderDate
  JOIN gold.DimCustomer dc ON dc.BusinessCustomerId = s.CustomerId
  JOIN gold.DimProduct dp ON dp.BusinessProductId = s.ProductId
  LEFT JOIN gold.DimSalesRep dr ON dr.BusinessSalesRepId = s.SalesRepId;
  ```

## Analytics (sample questions)
- **Customer behavior**: Repeat purchases, churn risk, cohort retention
- **Product performance**: Top products by revenue/volume, category contribution, cannibalization
- **Sales trends**: Period-over-period growth, seasonality, forecast baseline

Example KPI query (MoM revenue growth):
```sql
WITH rev AS (
  SELECT d.Year, d.Month, SUM(f.Amount) AS Revenue
  FROM gold.FactSales f
  JOIN gold.DimDate d ON d.DateKey = f.DateKey
  GROUP BY d.Year, d.Month
)
SELECT r2.Year, r2.Month,
       r2.Revenue,
       (r2.Revenue - r1.Revenue) / NULLIF(r1.Revenue, 0.0) AS MoM_Growth
FROM rev r2
LEFT JOIN rev r1
  ON (r1.Year = CASE WHEN r2.Month = 1 THEN r2.Year - 1 ELSE r2.Year END
      AND r1.Month = CASE WHEN r2.Month = 1 THEN 12 ELSE r2.Month - 1 END)
ORDER BY r2.Year, r2.Month;
```

## Performance Tips
- Add clustered columnstore index on large fact tables in Enterprise editions
- Add nonclustered indexes on foreign keys and frequent filters
- Use `DATE`/`INT` keys for efficient joins and partitioning
- Validate row counts across layers during ETL

## Data Quality Checks (examples)
- Uniqueness: business keys in dimensions
- Referential integrity: all fact foreign keys resolve to a dimension member
- Null/Range checks on critical attributes (e.g., negative amounts)

## Troubleshooting
- Data load fails: verify file path, UTF-8 encoding, delimiters, headers
- Type conversion errors: standardize datatypes in Silver before Gold
- Duplicates: apply survivorship rules and unique constraints in Silver

## License
MIT. See original repository for details.

## Credits
Based on the public project by `Abhayyyy0` (`SQL-data-warehouse`). This README adapts the approach for local use in SQL Server.
