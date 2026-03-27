# ✈️ Airline Analytics Dashboard

<div align="center">

![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?style=flat-square&logo=powerbi&logoColor=black)
![Power Query](https://img.shields.io/badge/Power%20Query-M%20Language-185FA5?style=flat-square)
![DAX](https://img.shields.io/badge/DAX-Measures-E74C3C?style=flat-square)
![Data Model](https://img.shields.io/badge/Data%20Model-Star%20Schema-27AE60?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-27AE60?style=flat-square)

**An interactive Power BI dashboard analysing US airline on-time performance across 200,000 flights — built on a star schema data model with all transformations handled natively in Power Query.**

 [Data Model](#-data-model) • [Dashboard Pages](#-dashboard-pages) • [DAX Measures](#-dax-measures)

</div>

---

## 📋 Overview

This project covers the full Power BI development lifecycle — data modelling, transformation, measure writing, and dashboard design — using US airline flight data for the full year 2023.

| What | Detail |
|---|---|
| Tool | Power BI Desktop |
| Transformations | Power Query (M language) — all built natively in Power BI |
| Data model | Star schema — 4 dimension tables, 7 fact tables |
| Measures | DAX — calculated KPIs that respond to slicers |
| Dashboard | 4 report pages, cross-filtered via airline and quarter slicers |
| Dataset | 200,000 flights, 8 airlines, 15 airports, full year 2023 |

---

## 📊 Dashboard Pages

### Page 1 — Executive Overview
Answers: *How are airlines performing overall?*

- 4 KPI cards — on-time rate, cancellation rate, avg departure delay, total flights — all built as DAX measures so they respond to slicers
- Line chart showing monthly on-time rate trend by airline across 12 months
- Horizontal bar chart ranking all 8 airlines by on-time performance with conditional colour formatting
- Airline slicer and quarter slicer connected to all visuals on the page

### Page 2 — Delay Analysis
Answers: *What is causing delays and when?*

- Stacked bar chart breaking down delay minutes by cause (Carrier, Weather, NAS, Late Aircraft, Security) per airline
- Donut chart showing overall delay cause split across all airlines
- Matrix heatmap showing delay rate by departure time period (rows) and day of week (columns) — conditional background colour formatting from white to red

### Page 3 — Routes & Airports
Answers: *Which routes and airports are busiest and most reliable?*

- Bubble map with all 15 airports plotted by lat/lon, bubble size driven by total departures
- Top 20 routes table filtered using Top N filter, with data bars on the on-time % column
- Clustered column chart showing on-time rate per airport with conditional colour (green / amber / red thresholds)

### Page 4 — Cancellations
Answers: *Why are flights being cancelled and which airlines cancel most?*

- Donut chart breaking cancellations into 4 reasons — Carrier, Weather, National Air System, Security
- Stacked horizontal bar showing cancellations per airline split by reason
- Line chart tracking monthly cancellation trend across all airlines throughout the year

---

## 🗄 Data Model

A star schema built entirely inside Power BI using Power Query. All surrogate keys are integer-based for fast relationship resolution.

```
                        Dim_Date
                       (date_key PK)
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
   Fact_Airline_    Fact_Delay_    Fact_Cancel-
   Monthly          Causes         lations
   (date_key FK)    (date_key FK)  (date_key FK)
   (airline_key FK) (airline_key FK)(airline_key FK)
              │             │             │
              └─────────────┼─────────────┘
                            │
                        Dim_Airline
                       (airline_key PK)


   Dim_Airport ──────► Fact_Airport_Performance
   (airport_key PK)    (airport_key FK)

   Dim_Route ─────────► Fact_Route_Performance
   (route_key PK)       (route_key FK)

   Fact_KPI_Summary     ← standalone (not slicer-filtered by design)
   Fact_Time_of_Day     ← standalone (not slicer-filtered by design)
```

### Dimension tables

| Table | Primary key | Rows | Contains |
|---|---|---|---|
| `Dim_Date` | `date_key` (YYYYMM) | 12 | Year, quarter, month, month name |
| `Dim_Airline` | `airline_key` | 8 | Carrier code, full airline name |
| `Dim_Airport` | `airport_key` | 15 | Code, city, state, latitude, longitude |
| `Dim_Route` | `route_key` | 210 | Origin, destination, route label, avg distance |

### Fact tables

| Table | Foreign keys | Rows | Contains |
|---|---|---|---|
| `Fact_Airline_Monthly` | date, airline | 96 | Monthly on-time %, delay rate, cancellation rate, avg delay |
| `Fact_Delay_Causes` | date, airline | 480 | Delay minutes broken out by cause (long format) |
| `Fact_Cancellations` | date, airline | 354 | Cancellations by reason per airline per month |
| `Fact_Route_Performance` | route | 210 | On-time %, delay rate, avg delay per route |
| `Fact_Airport_Performance` | airport | 15 | Departure volume, on-time %, delay rate per airport |
| `Fact_KPI_Summary` | — | 8 | Full-year static KPI values |
| `Fact_Time_of_Day` | — | 35 | Delay rate by time period and weekday |

---

## ⚙️ Power Query Transformations

All transformations are built natively in Power Query (M language) inside Power BI — no external preprocessing. The queries run in this order:

```
Raw_Flights ──────┐
Raw_Airlines ─────┤──► Silver_Flights (clean + enriched)
Raw_Airports ─────┘         │
                             ├──► Dim_Date
                             ├──► Dim_Airline
                             ├──► Dim_Airport
                             ├──► Dim_Route
                             ├──► Fact_Airline_Monthly
                             ├──► Fact_Delay_Causes
                             ├──► Fact_Cancellations
                             ├──► Fact_Route_Performance
                             ├──► Fact_Airport_Performance
                             ├──► Fact_Time_of_Day
                             └──► Fact_KPI_Summary
```

### Key transformations in Silver_Flights

- Set correct data types on all 21 raw columns
- Remove invalid rows where origin equals destination
- Fill null values in delay cause columns with 0
- Extract date parts — year, month, quarter, day of week, day name
- Parse departure time integer into HH:MM format and time period label
- Add delay flags — `is_delayed` (dep_delay > 15 min), `is_on_time`
- Add delay severity bucket — On Time / Minor / Moderate / Severe
- Add cancellation reason label from cancellation code
- Add route label (origin → destination)
- Add `date_key` surrogate (YYYYMM integer) for star schema join
- Left join airline names from `Raw_Airlines`
- Left join origin airport city, state, lat, lon from `Raw_Airports`
- Left join destination airport city, state, lat, lon from `Raw_Airports`

### Key transformations per fact table

- `Fact_Airline_Monthly` — grouped by date + airline, aggregated flight counts and delay metrics, joined `airline_key` from `Dim_Airline`
- `Fact_Delay_Causes` — filtered to delayed flights only, grouped and unpivoted cause columns to long format using `Table.UnpivotOtherColumns`
- `Fact_Route_Performance` — grouped by origin + dest, filtered to routes with 50+ flights using `Table.SelectRows`, joined `route_key` from `Dim_Route`

---

## 📐 DAX Measures

All KPI cards on Page 1 are DAX measures built on `Fact_Airline_Monthly`. Because this table is connected to both `Dim_Date` and `Dim_Airline` via relationships, the measures respond automatically to slicers without any extra filter logic.

```dax
On-Time Rate % =
DIVIDE(
    SUM(Fact_Airline_Monthly[on_time]),
    SUM(Fact_Airline_Monthly[total_flights])
) * 100
```

```dax
Cancellation Rate % =
DIVIDE(
    SUM(Fact_Airline_Monthly[cancelled]),
    SUM(Fact_Airline_Monthly[total_flights])
) * 100
```

```dax
Avg Dep Delay (min) =
DIVIDE(
    SUMX(
        Fact_Airline_Monthly,
        Fact_Airline_Monthly[avg_dep_delay] * Fact_Airline_Monthly[delayed]
    ),
    SUMX(
        Fact_Airline_Monthly,
        IF(Fact_Airline_Monthly[avg_dep_delay] > 0, Fact_Airline_Monthly[delayed], 0)
    )
)
```

```dax
Total Flights =
SUM(Fact_Airline_Monthly[total_flights])
```

The avg departure delay uses a **weighted average** — multiplying each row's average delay by its delayed flight count before dividing. This ensures the measure shifts meaningfully when filtered to a specific airline or quarter, rather than staying flat near the overall average.

---

## 🔧 Key Design Decisions

**Why build transformations in Power Query instead of loading pre-processed files?**
Building natively in Power Query means the report refreshes correctly if the source data ever changes — the entire cleaning and modelling pipeline reruns automatically. It also keeps all logic in one place rather than split across external tools.

**Why a star schema instead of a single flat table?**
One flat table with 200K rows works for basic charts but makes slicers slow and creates ambiguous aggregations in DAX. The star schema means a single slicer click on `Dim_Airline` filters `Fact_Airline_Monthly`, `Fact_Delay_Causes`, and `Fact_Cancellations` simultaneously through relationships — no extra DAX filter context needed.

**Why is date_key an integer (202301) instead of a date column?**
The data is at monthly grain — there is no meaningful day-level detail. An integer YYYYMM key is human-readable, sorts correctly, joins in milliseconds, and avoids the complexity of a full date table while still enabling quarter and month slicing.

**Why are Fact_KPI_Summary and Fact_Time_of_Day standalone with no relationships?**
Connecting them to dimensions would let slicers distort their values. The KPI summary is intentionally full-year totals — filtering it by airline would give misleading headline numbers. The time-of-day heatmap shows the overall pattern across all data — filtering it by month would produce a sparse, unrepresentative grid.

---

## 📦 Dataset

Synthetic data mirroring the schema of the Bureau of Transportation Statistics (BTS) On-Time Performance dataset.

| Property | Value |
|---|---|
| Total flights | 200,000 |
| Airlines | 8 — American, Delta, United, Southwest, JetBlue, Alaska, Spirit, Frontier |
| Airports | 15 major US hubs |
| Routes | 210 origin-destination pairs |
| Date range | January – December 2023 |
| Overall on-time rate | 86.2% |
| Overall cancellation rate | 2.0% |
| Top delay cause | Carrier (34.8% of all delay minutes) |

---

## 📄 License

MIT License — free to use, adapt, and build on.

---

<div align="center">
Built as a portfolio project demonstrating end-to-end Power BI development.<br>
If you found this useful, consider giving it a ⭐
</div>
