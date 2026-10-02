# Logistics Operations — Profitability Analysis (Power BI)

**Author:** Anthony Oko (Tonyvic)
**Tools:** Power BI Desktop (Power Query, Data Modelling, DAX)
**Project type:** TS Academy Data Analytics Capstone (open-ended)

---

## Executive Summary

This project analyzes three years (2022–2024) of operational data from a logistics/trucking company to answer one core question: **the business is profitable on paper, but where specifically is that profit being made — and where is it quietly being lost?**

Using a 14-table relational dataset covering drivers, trucks, routes, loads, fuel, maintenance, and safety records, I built a 3-page Power BI dashboard that traces revenue and cost down to the route, customer, and booking-type level — while being careful not to misrepresent costs that can't honestly be allocated that granularly (see *Data Modelling Decisions* below).

**Headline numbers:**

| Metric | Value |
|---|---|
| Total Revenue | $298,621,428.94 |
| Total Cost (Fuel + Maintenance + Safety) | $103,976,737.14 |
| Profit | $194,644,691.80 |
| Profit Margin | 65.18% |
| Total Trips | 85,410 |
| Fleet Avg MPG | 6.45 |
| On-Time Rate (Delivery leg) | 44.61% |

---

## Business Problem

> We're profitable on paper, but we don't know which parts of the business are actually earning that profit and which parts are quietly eating into it.

**Business questions addressed:**
1. Which routes are genuinely profitable once fuel cost is factored in?
2. Which customers generate the most profit, and which look good on revenue but are costly to serve?
3. Does booking type (Dedicated / Contract / Spot) affect actual profitability?
4. How much does maintenance cost erode margin, and does it vary by truck?
5. What's the real cost of safety incidents, and which incident types drive it?
6. Does service failure (late deliveries, detention) carry a cost consequence?

---

## Dataset

14 CSV tables: `customers`, `drivers`, `facilities`, `trucks`, `trailers`, `routes`, `loads`, `trips`, `fuel_purchases`, `maintenance_records`, `delivery_events`, `safety_incidents`, `driver_monthly_metrics`, `truck_utilization_metrics`.

- 85,410 loads / trips, 2022–2024
- 200 customers, 150 drivers, 120 trucks, 58 routes
- Referential integrity confirmed clean across all 14 tables (zero orphaned foreign keys)

---

## Data Preparation (Power Query)

- Verified a duplicate `fuel_purchases.xlsx` was byte-for-byte identical to `fuel_purchases.csv` (same 196,442 rows, same values) before excluding it from the model — importing both would have doubled every fuel cost figure.
- Confirmed CSV date columns were parsed under the correct locale (day-first, matching source format) rather than trusting auto-detection blindly.
- Enriched `routes` with full state names (`origin_state_full`, `destination_state_full`) via a merged lookup table, for management-readable labels.

---

## Data Modelling Decisions

- Built a dedicated `Date` table (Jan 2022–Jan 2025) and connected it to every transactional table.
- **Fixed a gap left by Power BI's relationship auto-detect:** `trips` had no relationships to `drivers`, `trucks`, or `trailers` despite valid matching keys — found by auditing the full relationship list rather than trusting the model diagram, and corrected manually.
- **Maintenance cost grain limitation:** `maintenance_records` only relates to `trucks`, not to `loads` or `routes` (a truck serves many routes/customers over its life, so a maintenance dollar can't be honestly attributed to one route). Reusing the full `Profit` measure in a route- or customer-sliced visual would silently apply the *entire* company's maintenance cost to every single row — confirmed this exact bug live before building around it. Solution: a separate `Profit (Excl. Maintenance)` measure is used on the Route & Customer Profitability page; maintenance cost is analyzed only at its correct grain (fleet/truck level) on the Cost Drivers & Risk page.
- **Fuel purchase and safety incident dates differ from the load's dispatch date** (0–3 day spread, confirmed in data — not assumed). Correct month-by-month trends for these use `USERELATIONSHIP()` in DAX to reference the true purchase/incident date rather than the load date.

---

## Key Insights

- **Fuel dominates controllable cost** — 91.94% of total cost ($95.59M of $103.98M), dwarfing maintenance (5.51%) and safety (2.55%).
- **Deliveries lag pickups badly on timeliness** — 44.61% on-time for the delivery leg vs. 66.73% for pickup. The service problem is concentrated on one side of the shipment, not spread evenly.
- **Booking type barely affects margin**, despite expectations — Spot (67.43%), Contract (67.07%), and Dedicated (66.95%) margins are nearly identical. Profitability is driven by lane and customer mix, not contract type.
- **Revenue is healthily diversified** — the single largest customer accounts for only 3.48% of total revenue; no dangerous concentration risk.
- **Equipment Damage is the costliest safety incident type** ($740,970.73) despite not being the most frequent (35 of 170 incidents) — DOT Violations are more frequent (39) but cheaper per incident.
- **Nearly a quarter of the fleet sits idle** — 28 of 120 trucks are in Maintenance or Inactive status.

---

## Problems Encountered & Solutions

| Problem | How it was found | Solution |
|---|---|---|
| `fuel_purchases` was uploaded twice (CSV + XLSX) | Compared row counts, IDs, and a full cell-by-cell diff | Confirmed identical; used the CSV only |
| CSV date columns risked being misread (ambiguous DD/MM vs MM/DD) | Checked a day-of-month value >12 to confirm correct parsing locale | Verified locale explicitly on every date column rather than trusting auto-detect |
| `trips` had no relationship to `drivers`, `trucks`, or `trailers` | Audited the full Manage Relationships list rather than reading the model diagram | Manually added the three missing many-to-one relationships; verified key formats matched first |
| Reusing the main `Profit` measure on route/customer visuals silently applied the whole company's maintenance cost to every row | Built a test table comparing `[Profit]` vs a maintenance-free measure side by side; the match was exact ($296,367.30 − $5,730,573.28 = −$5,434,205.98) | Built dedicated `Profit (Excl. Maintenance)` measures for any visual sliced below the fleet level |
| `safety_incidents.claim_amount` looked like it might double-count `vehicle_damage_cost` + `cargo_damage_cost` | Checked the relationship between the three columns directly | Confirmed `claim_amount` **is** the sum of the other two — used `claim_amount` alone as Total Safety Cost, avoiding a 2x overstatement |
| Fuel/safety cost trends would have used the wrong date (load date instead of actual purchase/incident date) | Compared `purchase_date`/`incident_date` to the load's date directly; found only ~30% matched | Used `USERELATIONSHIP()` in DAX for any month-level fuel/safety trend |
| A booking-type donut chart showed misleading percentages after adding multiple measures to one visual | Noticed the percentages didn't match any known total; traced them to a combined (and meaningless) sum of revenue + profit | Limited the donut to one measure only, moved supporting figures to tooltips |
| `fuel_purchases` query failed to refresh ("failed to move the data reader to the next row") after a data-type change on a 196K-row table | Isolated the fault to one table by checking which visuals broke vs. stayed working | Closed without saving to revert safely, then redid the fix one table at a time with a stable power/connection before retrying on the large table |

---

## Project Files

- `README.md`
- `Logistics_Capstone.pbix` — full Power BI report
- `Overview.png`, `RouteCustomerProfitability.png`, `CostDriversRisk.png` — dashboard screenshots

---

## Learning Outcomes

- Practiced auditing a relationship model directly (Manage Relationships list) rather than trusting the visual diagram, and caught a real auto-detect gap as a result.
- Learned to recognize when a measure's *grain* doesn't match the dimension it's being sliced by, and to design around that honestly instead of forcing a misleading allocation.
- Used `USERELATIONSHIP()` to handle a table with more than one valid date relationship.
- Learned that pie/donut charts can hide a multi-measure mixing error that a bar chart would expose — verified chart output against source data rather than trusting a visual that "looked" fine.

---

## Author

Anthony Oko (Tonyvic)
GitHub: [github.com/Tonyvicoko](https://github.com/Tonyvicoko)
