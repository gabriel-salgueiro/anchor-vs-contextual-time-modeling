# Anchor vs Contextual Time Modeling Framework
### Comparing fixed calendar-based intelligence with user-defined contextual modeling
> Built with disconnected date modeling and dynamic metric parameterization

---

## Overview

Most Power BI time intelligence implementations rely on built-in DAX functions like `SAMEPERIODLASTYEAR`, `DATEADD`, and `DATESYTD`. These work well for standard calendar-based comparisons: MoM, YoY, YTD and cover the majority of analytical use cases.

However, in certain business contexts, particularly **retail**, where promotional actions, campaign windows, and tactical events don't align with fixed calendar intervals, these functions fall short. Comparing a 10-day flash sale in March against the same 10 days in February, or evaluating a promotional tabloid from week 3 of one month against week 2 of another, requires a different approach.

This project demonstrates both models side by side, with no judgment on which is "better": the goal is to show when each approach delivers the most value.

---

## Models

### Standard Time Intelligence (Predefined Comparisons)

Uses native DAX time intelligence functions anchored to a user-selected Month/Year. All metrics are evaluated relative to this anchor month.

| Column | Description |
|---|---|
| YTD (CY) | Jan of selected year → End of anchor month |
| YTD (PY) | Jan of prior year → Same month (prior year) |
| Month (CM) | Anchor month value |
| Month (PM) | Previous month value |
| Δ MoM | Absolute variance: CM vs PM |
| MoM % | Percentage variance: CM vs PM |
| Same Month LY | Same month, prior year |
| Δ | Absolute variance: CM vs Same Month LY |
| Month YoY % | Percentage variance: CM vs Same Month LY |

The subtitle dynamically communicates both contexts simultaneously:
```
Anchor month: Dec 2025 | YTD (CY): Jan 01, 2025 → Dec 31, 2025 | YTD (PY): Jan 01, 2024 → Dec 31, 2024
```

### Flexible Period Comparison (User-Defined Contexts)

Uses **disconnected date tables** to allow arbitrary period pairing, completely independent of the calendar structure. The user defines a Start Date range and an End Date range, and the model computes the comparison between them.

| Column | Description |
|---|---|
| Start | Metric value for the Start Date period |
| End | Metric value for the End Date period |
| Δ | Absolute variance |
| % Change | Percentage variance with directional indicator |

**Predefined periods** are available for quick selection:
- **MoM** - Previous month vs Current month
- **Same Month LY** - Same month, prior year
- **YoY (Full Year)** - Full year 2024 vs Full year 2025
- **Custom** - Any arbitrary date range, day-level precision

---

## Technical Implementation

### Disconnected Date Tables

The core of the flexible model. Two separate date tables: `_Start Date` and `_End Date` have **no active relationship** with the fact table. Instead, metrics are calculated using `USERELATIONSHIP` inside `CALCULATE`, activated only at measure evaluation time.

```dax
Start Date Metric Value = 
VAR minDate = [Selected Start Date MIN]
VAR maxDate = [Selected Start Date MAX]
RETURN
CALCULATE(
    [Metric Value],
    USERELATIONSHIP(fSales[Order_Date], '_Start Date'[Start Date]),
    '_Start Date'[Start Date] >= minDate,
    '_Start Date'[Start Date] <= maxDate
)
```

This pattern allows the user to filter any two arbitrary date ranges simultaneously, something impossible with a single active calendar relationship.

### Context-Aware Measures

A critical challenge with the standard DAX time intelligence functions is **filter context dependency**. Without an explicit year filter active, measures like `SAMEPERIODLASTYEAR` return incorrect results because the base measure aggregates across all years.

The solution uses `FILTER(ALL(_dimCalendar), ...)` to explicitly define the evaluation context, making measures fully independent of external slicers:

```dax
Metric MoM % = 
VAR lastYear = YEAR(CALCULATE(MAX(_dimCalendar[Date])))
VAR lastMonth = MONTH(CALCULATE(MAX(_dimCalendar[Date])))
VAR prevYear = IF(lastMonth = 1, lastYear - 1, lastYear)
VAR prevMonth = IF(lastMonth = 1, 12, lastMonth - 1)
VAR currentValue = CALCULATE(
    [Metric Value],
    FILTER(ALL(_dimCalendar), _dimCalendar[Year] = lastYear && _dimCalendar[Month Number] = lastMonth)
)
VAR prevValue = CALCULATE(
    [Metric Value],
    FILTER(ALL(_dimCalendar), _dimCalendar[Year] = prevYear && _dimCalendar[Month Number] = prevMonth)
)
RETURN
IF(ISBLANK(prevValue), BLANK(), DIVIDE(currentValue, prevValue, 0) - 1)
```

### Predefined Period Selection

A disconnected `_Periods` table drives the quick-select slicer. Four measures read the selection and return the appropriate date boundaries, falling back to the custom slicers when "Custom" is selected:

```dax
Selected Start Date MIN = 
VAR sel = SELECTEDVALUE(_Periods[Period Label], "Custom")
RETURN
SWITCH(
    sel,
    "MoM",            DATE(2025, 11, 1),
    "Same Month LY",  DATE(2024, 12, 1),
    "YoY (Full Year)", DATE(2024, 1, 1),
    MIN('_Start Date'[Start Date])
)
```

### Dynamic Metric Parameterization

A `Metric` table drives a single slicer that controls all visuals simultaneously. Dynamic format strings adapt formatting per metric type without requiring separate formatted measures:

```dax
-- Dynamic format string (applied directly to measure)
VAR m = SELECTEDVALUE(Metric[Metric], "Revenue")
RETURN
SWITCH(m, "Orders", "#,0", "AOV", "$#,0.00", "$#,0")
```

---

## Dataset

Synthetic retail dataset with 20,065 rows covering January 2024 through December 2025.

- **2024:** 10,000 original orders
- **2025:** ~10,065 generated orders with realistic YoY variation

**2025 generation methodology:** Each 2024 order was replicated with the year shifted to 2025. Category-level growth factors and monthly volume multipliers were applied to introduce realistic variance, both in revenue/profit values and in order volume per month.

| Category | Revenue Growth |
|---|---|
| Beauty | +4% |
| Clothing | +2% |
| Home & Kitchen | +3% |
| Electronics | -3% |
| Sports | -2% |

Monthly order volume varies independently of revenue growth, breaking the mirror effect between years and adding analytical credibility to MoM comparisons.

**Columns:** Order_ID, Customer_ID, Order_Date, Region, Product_Category, Customer_Segment, Quantity, Unit_Price, Discount_Rate, Revenue, Cost, Profit, Payment_Method

---

## Key Takeaways

- Standard time intelligence functions are not wrong, they are the right tool for the majority of use cases
- Disconnected date tables unlock period flexibility that calendar-based functions cannot provide
- The value of the flexible model is highest in contexts with **abstract or irregular promotional cadences**: retail, marketing, campaign analysis
- DAX filter context must be explicitly managed when measures need to be independent of external slicers: `FILTER(ALL(...))` is more reliable than `KEEPFILTERS` for this pattern

---

## Stack

- Power BI Desktop
- DAX
- Python (dataset generation: pandas, numpy)
