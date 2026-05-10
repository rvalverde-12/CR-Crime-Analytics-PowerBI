# Costa Rica Public Security Analytics (OIJ) - Comprehensive Portfolio Overview

## Project Overview
This project transforms raw national crime data from the Organismo de Investigación Judicial (OIJ) into an interactive, geospatial Power BI dashboard. Designed as a strategic intelligence tool, it enables stakeholders to analyze spatiotemporal crime patterns, assess regional risk multipliers, and profile victim demographics between 2024 and 2026.

## The Business Problem
Raw incident logs fail to provide immediate, actionable intelligence. Policymakers and law enforcement struggle to allocate resources efficiently without knowing the exact geographic hotspots, peak timeframes, and vulnerable demographics for specific crime types (e.g., vehicle theft vs. assault). 

##  The Solution
A dynamic, interactive reporting suite that provides:
1. **Geospatial Hotspotting:** Interactive mapping to identify high-density incident areas down to the Canton level.
2. **Temporal Risk Matrices:** A 12-hour window heatmap to pinpoint exact days and times when specific crimes spike.
3. **Risk Multiplier Analysis:** A scatter plot matrix comparing raw incident volume against a calculated comparative risk score.
4. **Demographic Profiling:** Clustered bar charts isolating victim age groups and gender for targeted public safety campaigns.

## Technical Stack & Skills Demonstrated
* **Database / Backend:** PostgreSQL (Connecting to local host, querying millions of rows).
* **Data Transformation:** Power Query (M Code) for data cleaning, custom conditional columns, and breaking DAX circular dependencies.
* **Data Modeling:** Built a robust Star Schema (Fact and Dimension tables) to ensure optimized filtering and cross-highlighting.
* **DAX Formulas:** Developed custom measures for volume calculations and created logical sort-keys to override default alphabetical sorting in visuals.
* **UI/UX Design:** Implemented modern dashboard design principles, including hidden bookmark-driven reset buttons, consistent Z-pattern layouts, and dynamic KPI ribbons.

---

##  Deep Dive: Data Extraction (SQL)
The foundation of the dashboard relies on a PostgreSQL database. Below are examples of the SQL queries utilized to extract and aggregate the raw data before importing it into Power BI.

### Extracting Core Incident Logs
```sql
SELECT 
    id,
    CAST(fecha AS DATE) AS date_of_incident,
    hora AS time_of_incident,
    UPPER(delito) AS crime_type,
    UPPER(subdelito) AS crime_subtype,
    provincia AS province,
    canton,
    distrito AS district,
    sexo AS victim_gender,
    edad AS victim_age_group
FROM 
    public.crime_stats
WHERE 
    fecha >= '2024-01-01'
    AND estado = 'CONFIRMADO';
```

### Calculating the "Relative Risk Multiplier"
To feed the scatter plot matrix, a query was designed to calculate a comparative risk score per Canton against the national average.
```sql
WITH CantonTotals AS (
    SELECT canton, provincia, COUNT(*) as actual_incidents
    FROM public.crime_stats
    GROUP BY canton, provincia
),
NationalAvg AS (
    SELECT AVG(actual_incidents) as avg_incidents
    FROM CantonTotals
)
SELECT 
    c.canton,
    c.provincia,
    c.actual_incidents,
    ROUND((c.actual_incidents * 1.0 / n.avg_incidents), 2) AS risk_multiplier
FROM 
    CantonTotals c
CROSS JOIN 
    NationalAvg n
ORDER BY 
    risk_multiplier DESC;
```

---

## Deep Dive: Data Transformation (Power Query & DAX)
To ensure the dashboard performed optimally and avoided sorting errors, complex transformations were pushed to the back-end (Power Query).

### Breaking the Circular Dependency (Power Query - M Code)
When attempting to sort categorical age groups by volume using DAX, a circular dependency occurred. The solution was migrating the logic to Power Query to create a static index before the data loaded:
```powerquery
#"Added Age Sort" = Table.AddColumn(#"Previous_Step", "Age_Sort", each 
    if [edad] = "Menor de edad" then 1 else 
    if [edad] = "Adulto Mayor" then 2 else 
    if [edad] = "Desconocido" then 3 else 
    if [edad] = "Mayor de edad" then 4 else 5, 
    Int64.Type)
```

### DAX Measures
Instead of relying on Power BI's "Implicit Measures", official DAX measures were created to act as the single source of truth:
```dax
Total Crimes = COUNT('public crime_stats'[id])
```

---

## Visualizations & Architecture Decisions
* **Filled Map (Grayscale Theme):** Addressed the "Area vs. Volume" illusion, ensuring users understand that large rural cantons visually dominate the map, but dense, small cantons hold the true volume. Muted the background to make data polygons pop.
* **Temporal Risk Matrix:** Applied custom sorting via Power Query to ensure days run logically (Monday to Sunday) and times progress chronologically, with conditional formatting tied to the `Total Crimes` measure.
* **Scatter Plot Matrix:** Set Y-axis aggregation to `Average` (rather than Sum) to prevent the "multiplication effect" where Power BI inflates multipliers based on row counts.
* **Demographic Profile:** Selected a *Clustered* Bar Chart over a *Stacked* Bar chart for a shared baseline, making it easier to compare secondary demographics (e.g., minors vs. senior citizens). Filtered out "Desconocido" (Unknown) to improve the signal-to-noise ratio.
* **Dynamic KPI Ribbon & Reset Button:** Built a top ribbon for instant insights (Total Crimes, Highest Risk Province, Most Frequent Crime). Implemented a Power BI `Bookmark` bound to a customized "Clear All" action button for seamless UX navigation.

---

## Dashboard Preview

---

