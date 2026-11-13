# HR Overview Dashboard — portfolio review

Reviewed 2026-11-12. Read-only: nothing in the model or report was changed.

Scope: `hrdashboard.pbip` (PBIP, report version 2.0.0, compatibility level 1606),
inspected both as TMDL/JSON on disk and as the live model in Power BI Desktop
(`localhost:58256`) over the Power BI modelling MCP server.

---

## 1. Summary

A four-page HR dashboard over the well-known fictional HR training dataset
(311 employees, 2006–2019). Five CSVs and one Excel workbook feed an eleven-table
model with 39 measures, a hand-built date table, a field parameter, a what-if
parameter, and a calculated bridge table. The report layer is the strongest part:
a custom nav rail, 14 bookmarks, a custom theme, SVG-in-DAX progress bars, a
field-parameter slicer and 123 hand-set interaction overrides. Somebody clearly
knew what they were doing in the report canvas.

The model underneath has not had the same care. The headline problems are that
**it cannot be refreshed by anyone except its original author** (every query points
at `C:\Users\Admin\Downloads\hrdashboard-main\`), that the **KPI row mixes two
different populations** (headcount counts only active staff, average salary and the
three age cards include the 104 people who have left), and that the date
intelligence runs through Power BI's **auto date/time tables** rather than the date
table the project went to the trouble of building. Tabular Editor's Best Practice
Analyzer reports **270 violations, 36 of them severity 3**.

Against a strong portfolio standard: the visual design and interaction work are
above average, the modelling discipline and the documentation are below it. The
good news is that the most damaging items are also the cheapest to fix.

---

## 2. Data and loading

### Source files

All six sources are flat files loaded with hard-coded absolute paths:

| Query | Source | Path in the model |
|---|---|---|
| `employees` | `employees.csv` | `C:\Users\Admin\Downloads\hrdashboard-main\employees.csv` |
| `employee_fact` | `employee_fact.csv` | `C:\Users\Admin\Downloads\hrdashboard-main\employee_fact.csv` |
| `departments` | `departments.csv` | `C:\Users\Admin\Downloads\hrdashboard-main\departments.csv` |
| `managers` | `managers.csv` | `C:\Users\Admin\Downloads\hrdashboard-main\managers.csv` |
| `positions` | `positions.csv` | `C:\Users\Admin\Downloads\hrdashboard-main\positions.csv` |
| `Photos` | `people_image_urls_and_ids.xlsx` | `C:\Users\Admin\Downloads\hrdashboard-main\people_image_urls_and_ids.xlsx` |

All six files also sit in this project folder, next to the `.pbip`. The paths
point at a **different Windows user profile** (`Admin`) and a **different folder
name** (`hrdashboard-main`), so a refresh fails for anybody who clones this. There
are **no Power Query parameters and no named expressions** in the model, so there
is no single place to change it either.

The data is fictional training data, so there is nothing to check on the privacy
side. Worth noting separately: `Photos` holds 311 avatar URLs on
`xsgames.co`, a free third-party avatar service. The Employees Breakdown page
depends on that host still serving images.

### Tables

| Table | Rows | Columns | Kind |
|---|---|---|---|
| `employees` | 311 | 21 (19 imported + 2 calculated) | people dimension, also carries flattened attributes |
| `employee_fact` | 311 | 18 | fact, one row per employee |
| `departments` | 6 | 2 | dimension (CSV has 8 rows) |
| `managers` | 22 | 2 | dimension (CSV has 24 rows) |
| `positions` | 29 | 2 | dimension (CSV has 34 rows) |
| `DateTable` | 5,164 | 1 | calculated calendar |
| `DepartmentsEmployees` | 311 | 6 | calculated bridge/summary |
| `Photos` | 311 | 3 | avatar URLs |
| `_DAX_measures` | 0 | 0 | measure container (38 measures) |
| `Rates years  window` | 11 | 1 + 1 measure | what-if parameter, `GENERATESERIES(0,10,1)` |
| `_selectorPage1` | 2 | 3 | field parameter (Age range / Hire source) |

Plus four hidden auto date/time tables: `DateTableTemplate_d21bed01…` and three
`LocalDateTable_…` tables, generated because `__PBI_TimeIntelligenceEnabled = 1`.

The fact table is one row per employee, same grain as `employees`, so this is
really a single wide entity split in two rather than a star schema. BPA flags it
as a snowflake.

### Relationships (12)

| From | To | Cardinality | Direction | Active |
|---|---|---|---|---|
| `employees[EmpID]` | `employee_fact[EmpID]` | 1:* | **both** | yes |
| `employee_fact[DeptID]` | `departments[DeptID]` | *:1 | single | yes |
| `employee_fact[PositionID]` | `positions[PositionID]` | *:1 | single | yes |
| `employee_fact[ManagerID]` | `managers[ManagerID]` | *:1 | single | yes |
| `employees[DepartmentID]` | `DepartmentsEmployees[DepartmentID]` | ***:*** | **both** | yes |
| `employees[EmpID]` | `Photos[ID_upd]` | 1:* | **both** | yes |
| `employees[DateofHire]` | `DateTable[Date]` | *:1 | single | **no** (used via `USERELATIONSHIP`) |
| `employees[DateofTermination]` | `DateTable[Date]` | *:1 | single | **no** (used via `USERELATIONSHIP`) |
| `employees[Department]` | `departments[Department]` | *:1 | single | **no** — auto-detected leftover, never activated |
| `employees[DOB]` | `LocalDateTable_98223052…[Date]` | *:1 | single | yes (auto date/time) |
| `employees[LastPerformanceReview_Date]` | `LocalDateTable_6fe9d1f7…[Date]` | *:1 | single | yes (auto date/time) |
| `DateTable[Date]` | `LocalDateTable_08de63b5…[Date]` | *:1 | single | yes (auto date/time) |

Three bidirectional relationships and one many-to-many, in a model with 311 rows
in its largest table. None of them needs to be bidirectional for the queries the
report runs; they are the kind of thing that works at this size and then does not
scale.

The last row is the odd one: the hand-built `DateTable` has an **auto date/time
table of its own** hanging off its `Date` column.

### Date table

```dax
DateTable = CALENDAR(MIN(employees[DateofHire]), MAX(employees[LastPerformanceReview_Date]) + 365)
```

- Range: **2006-01-09 → 2020-02-28** (5,164 days).
- It has **exactly one column**, `Date`. No Year, Quarter, Month, Month Name,
  Day of Week, no hierarchy.
- It is **not marked as a date table** (`dataCategory: Time` is absent).

Because it has no Year or Month column, every time axis and every time-intelligence
measure in the report reaches for `DateTable[Date].[Year]` and
`DateTable[Date].[Month]` — the **auto date/time hierarchy**. Confirmed at query
time: a `SUMMARIZECOLUMNS(DateTable[Date].[Year], …)` comes back with the column
named `LocalDateTable_08de63b5-f2fa-404b-9be6-77cf88bb8898[Year]`.

So the project built a date table and then did not use it as one. If auto
date/time is ever switched off — which is the standard recommendation, and which
the BPA rule file asks for in four places — every trend chart and every rate
measure on the HR Metrics page breaks at once.

The `+ 365` is a second problem. The last real event in the data is a performance
review on **2019-02-28**; the last hire is 2018-07-09 and the last termination is
2018-11-10. The calendar runs a full year past that, to 2020-02-28. See §5.

### Measures (39)

38 live in `_DAX_measures`, one (`Rates years  window Value`) in the what-if table.
All but one have a display folder. **None of the 39 has a description.**

**Employee**

| Measure | What it does |
|---|---|
| `EmployeeTotalCurrent` | Distinct count of `EmpID` where `EmploymentStatus = "Active"`. |
| `EmployeeSalaryAVG` | `AVERAGE(employee_fact[Salary])` — no status filter. |
| `EmployeeAgeMIN` / `EmployeeAgeMAX` / `EmployeeAgeAVG` | Min / max / `ROUNDDOWN`-ed average of the `Age` calculated column. |
| `EmployeesCount_Left` | Distinct employees terminated in the period, via `USERELATIONSHIP` on `DateofTermination`. |
| `EmployeesCount_LeftVoluntarily` | Same, restricted to `"Voluntarily Terminated"`. |
| `EmployeesCount_StartOfPeriod` | Headcount at `FIRSTDATE` of the period (`FILTER` over the whole `employees` table). |
| `EmployeesCount_EndOfPeriod` | Same at `LASTDATE`. |
| `EmployeesCount_AvgPerPeriod` | Mean of start and end headcount. |
| `EmployeesHiredPerPeriod` | Hires in the max year of the current context. |
| `EmployeeTerminationDate` | Termination date as text, or `"N/A"`. |
| `EmployeeSatisfactio` | `5 - AVERAGE(EmpSatisfaction)` — the remainder half of a donut gauge (name is a typo). |
| `EmployeeEngagement` | `5 - AVERAGE(EngagementSurvey)` — same pattern. |
| `EmployeeSalaryAVGbyDep` | Department average salary, `ALLEXCEPT` on `DepartmentsEmployees[Department]`. |
| `EmployeesSalaryAVG vs DeptAVG` | Employee salary ÷ department average − 1. |
| `EmployeeSalary vs SelectedEmployee` | Employee salary ÷ `EmployeeSalaryAVG` − 1. |
| `EmployeeSalaryDiff vs DeptAVG %` | `SalaryMAX` ÷ `EmployeeSalaryAVGbyDep` − 1. |
| `EmployeeSelected` | Returns `"#E4E6C3"` when the row is the selected employee — conditional-formatting helper. |

**Salary** — all seven base measures wrap the aggregation in
`CALCULATE(…, USERELATIONSHIP(DateTable[Date], employees[DateofHire]))`, i.e. they
are "salary by hire date":

`SalaryMIN`, `SalaryPERC25`, `SalaryMEDIAN`, `SalaryAVG`, `SalaryPERC75`,
`SalaryMAX`, plus three derived spreads `SalaryIQR`, `Salary50-25`, `Salary75-50`
used to stack a box-plot-style column chart.

**Turnover / Attrition / Retention** — three parallel sets of three:

| Measure | What it does |
|---|---|
| `Turnover Rate` | Leavers ÷ average headcount; falls back to `AVERAGEX` over years when more than one date is in context. |
| `Attrition Rate` | Same shape, voluntary leavers only. |
| `Retention Rate` | `(End − Hired) ÷ Start`, same `AVERAGEX` fallback. |
| `… vs Prev Period` (×3) | Month-over-month if `ISINSCOPE` month, else year-over-year, via `DATEADD`. |
| `…RateLastYears` (×3) | The rate over the last *N* years, *N* from the what-if slicer. |

**Unfoldered**

| Measure | What it does |
|---|---|
| `EngagementRate ProgressBar` | Builds an SVG gradient bar as a data URI; returns `BLANK()` unless one employee is selected. |

### Calculated columns (2)

```dax
employees[Age] =
VAR maxYear = MAX(DateTable[Date].[Year])
VAR yearOfBirth = YEAR(employees[DOB])
RETURN maxYear - yearOfBirth
```

```dax
employees[Age ranges] =
SWITCH(TRUE(),
    employees[Age] < 25, "18-24",
    employees[Age] >= 25 && employees[Age] < 35, "25-34",
    … , employees[Age] >= 65, "65+")
```

Both are correctly columns rather than measures — they are row attributes used for
slicing and grouping, and `Age ranges` is one of the two field-parameter options.
Two caveats:

- `Age` resolves `MAX(DateTable[Date].[Year])` once, at refresh, over the whole
  calendar. That is **2020**, the padded end of the date table, not today and not
  the end of the data. So every age in the model is "age as of 2020", frozen.
  For a fixed training dataset that is defensible, but nothing anywhere says so.
- `Age` also reaches into the auto date/time hierarchy of `DateTable`, so it is a
  third dependency on auto date/time.
- The `"18-24"` bucket is empty: the youngest employee is 28 (youngest DOB is
  1992-08-17). The label would also be wrong for anyone under 18.

### Roles

**None.** No row-level security is defined. For a dashboard published with
Publish to web that is the right answer — RLS and Publish to web are mutually
exclusive — but an HR model is the textbook case for demonstrating RLS, so it is
a missed opportunity rather than a defect.

### Power Query steps that matter

**`employees`** — after the type change, it merges `employee_fact` twice,
`positions` once and `departments` once, in order to flatten `Position`,
`DepartmentID` and `Department` onto the employee row:

```
Merged Queries (employee_fact) → Expanded PositionID
→ Merged Queries1 (positions)  → Expanded Position
→ Merged Queries2 (employee_fact) → Expanded DeptID
→ Merged Queries3 (departments) → Expanded Department
```

Those flattened columns look like duplicates of `positions[Position]` and
`departments[Department]`, which are already reachable through relationships. The
report then uses *both*: the Overview donut groups on `departments[Department]`
while the Salary Distribution table shows `departments[Department]` and the
Employees Breakdown table shows `employees[Position]`. BPA flags
`employees[Employee_Name]` as redundant for the same reason.

> **Correction (applied later).** Calling these pure duplication was wrong, and
> the reason matters. `positions` and `departments` sit on the *one* side of
> their relationships to `employee_fact`, so a filter on a single employee never
> reaches them — filters travel one-side-to-many, not the reverse. That makes the
> flattened `employees[Position]` and `employees[Department]` the only way to
> read a selected person's job title and department, which is exactly what the
> Employees Breakdown profile needs. They are load-bearing, not redundant. See
> §8 for the bug this caused on the live page.

**`employee_fact`** — hand-coded key repairs:

```m
Table.ReplaceValue(…, null, 39, …, {"ManagerID"})
Table.SelectRows(…, each true)                      // no-op step
Table.Sort(…, {{"ManagerID", Order.Descending}})
Table.ReplaceValue(…, 30, 22, …, {"ManagerID"})
Table.ReplaceValue(…, 17, 18, …, {"PositionID"})
```

**`departments`** — remaps `Software Engineering → 4` and `Production → 5` through
an added `Custom` column, then drops and renames the original `DeptID`; 8 source
rows become 6.

**`positions`** — renames position 13 to `"IT Manager"`, remaps 17→18 and
`Software Engineer`→24, then dedupes; 34 source rows become 29.

**`managers`** — the same null→39 and 30→22 repairs, then dedupe; 24 → 22.

**`Photos`** — `Table.Skip(…, 11)` drops the first 11 rows with no comment.

These are all legitimate data-cleaning decisions, but every one of them is a magic
number with no explanation. On a portfolio piece that is the sort of thing an
interviewer will ask about, and there is no answer in the file.

---

## 3. Report

Four pages, all 1920×1080, `displayOption: None` (actual size).

| # | Page | Visibility | Containers |
|---|---|---|---|
| 1 | **Overview** | visible | 52 |
| 2 | **Salary Distribution** | hidden in view mode | 56 |
| 3 | **HR Metrics Overview** | hidden in view mode | 53 |
| 4 | **Employees Breakdown** | hidden in view mode | 62 |

223 containers in total. Pages 2–4 being hidden is **deliberate and correct**: the
left rail on every page carries `PageNavigation` buttons to all four, so the native
page tabs are suppressed in favour of the custom nav. Verified in the button
definitions (`navigationSection` targets `543e009b…`, `3765c215…`, `2bda0597…`).

**Report-level filters (2)**

1. `positions[Position]` **is not** `'President & CEO'` — inverted selection, so
   the CEO is excluded from every visual on every page.
2. `DateTable[Date].[Year]` **is not** blank.

Filter one changes the headline numbers (see §5) and is nowhere stated on the
canvas.

**Theme.** Base `CY24SU05` plus a registered custom theme
(`Custom623609004858712.json`): `callout` 14 pt Segoe UI Light, `title` 24 pt
Segoe UI, and an eight-colour data palette
`#222725 #899878 #E4E6C3 #CCCCCC #000000 #605E5C #E6E6E6 #FFFFFF` — a dark
green-grey, sage, pale lime, then neutrals.

Across the four pages: **303 `ThemeDataColor` references** against **89 literal
hex values** in 7 distinct colours. Six of the seven are palette colours written
out by hand; **two are off-palette**:

- `#29D1F2` (bright cyan) — a button `fillColor`, 2 uses per page on pages 1–3.
- `#F1BA62` (orange) — `decreaseFill` on the three waterfall charts on HR Metrics.

The cyan in particular has nothing to do with the sage/stone palette. The orange
is at least defensible as a semantic "down" colour, but a theme-level
`negative`/`positive` pair would do it properly.

**Bookmarks (14).** `open filter pane` / `close filter pane` (×3 pages),
`Show`/`Close Name selector`, and field-swap bookmarks `years`, `Years`,
`positions`, `Department`, `Performance`, `Work status`. Note `years` and
`Years` as two separate bookmarks differing only in case.

> **Correction (applied later).** An earlier draft of this review also listed
> `positions`/`Positions` as a case-duplicate pair. That was wrong: there is one
> bookmark named `positions`, and `Positions` is the *label on a button*, not a
> second bookmark. Only `years`/`Years` were a genuine duplicate.

**Visual interactions.** 123 explicit overrides, almost all `NoFilter`:
6 on Overview, 8 on Salary Distribution, **108 on HR Metrics Overview**, 1 on
Employees Breakdown. On HR Metrics every one of the nine data visuals is set not
to cross-filter the other eight — an all-pairs matrix. The effect is right (nine
independent rate charts) but it was built by hand, 108 clicks, and will need
redoing from scratch whenever a visual is added.

**Custom visual.** One: `simpleImage` (`simpleImageEBC4593F96F1425FB3D84C5BF02B5075`),
used on Employees Breakdown to render the employee avatar from `Photos[ImageURL]`.

### Page 1 — Overview

| Visual | Type | Fields | Title |
|---|---|---|---|
| KPI card | `card` | `[EmployeeTotalCurrent]` | "Employees count" |
| KPI card | `card` | `[EmployeeSalaryAVG]` | "Average salary" |
| KPI card | `card` | `[EmployeeAgeMIN]` | "Min age of employees" |
| KPI card | `card` | `[EmployeeAgeAVG]` | "Average age of employees" |
| KPI card | `card` | `[EmployeeAgeMAX]` | "Max age of employees" |
| Donut | `donutChart` | `employees[Sex]` × `[EmployeeTotalCurrent]` | "Employee count by Sex" |
| Combo | `lineStackedColumnComboChart` | `employees[RaceDesc]`, series `employees[Sex]`, Y `[EmployeeTotalCurrent]`, Y2 `[EmployeeSalaryAVG]` | "Employees count by sex and **ethic** group" |
| Bar | `barChart` | `employees[Position]` × `[EmployeeTotalCurrent]` | "Employees count by Position" |
| Bar (hidden) | `barChart` | `DateTable[Date].[Year]` × `[EmployeesCount_EndOfPeriod]` | "Employees count by year" — bookmark alternate |
| Donut | `donutChart` | `employees[MaritalDesc]` × `[EmployeeTotalCurrent]` | "Employee count by Marital Status" |
| Donut | `donutChart` | `departments[Department]` × `[EmployeeTotalCurrent]` | "Employee count by Department" |
| Matrix | `pivotTable` | rows `employees[RecruitmentSource]`, cols `employees[Sex]`, values `[EmployeeTotalCurrent]`, `[EmployeeSalaryAVG]` | none |
| Field slicer | `advancedSlicerVisual` | `_selectorPage1[_selectorPage1]` | none |

Slicers (7, in the collapsible left pane): `employees[Age]` (numeric range),
`employees[Sex]`, `employees[MaritalDesc]`, `employees[RaceDesc]`,
`departments[Department]`, `managers[ManagerName]`, `employees[RecruitmentSource]`.
Plus 16 shapes, 9 action buttons, 1 textbox, 1 image.

### Page 2 — Salary Distribution

Six KPI cards: `[SalaryMIN]`, `[SalaryPERC25]`, `[SalaryMEDIAN]`, `[SalaryAVG]`,
`[SalaryPERC75]`, `[SalaryMAX]`, titled "Min salary", "25 percentile",
"Median salary", "**Average saalary**", "75 percentile", "Max salary".

| Visual | Type | Fields | Title |
|---|---|---|---|
| Box-plot style | `lineStackedColumnComboChart` | `DateTable[Date].[Year]`; Y `[SalaryPERC25]` `[Salary50-25]` `[Salary75-50]`; Y2 `[SalaryMIN]` `[SalaryMEDIAN]` `[SalaryMAX]` `[SalaryAVG]` | "Salary distribution by years" |
| same, hidden | `lineStackedColumnComboChart` | `employee_fact[PerformanceScore]`, same measures | "Salary distribution by employees' performance" — bookmark alternate |
| Line | `lineChart` | `employee_fact[EmpSatisfaction]`, series `employees[EmploymentStatus]`, Y `[SalaryAVG]` | "Employee salary by Work status and Satisfaction rate" |
| Donut, hidden | `donutChart` | `departments[Department]` × `[SalaryAVG]` | "Employee salary by department" — bookmark alternate |
| Combo | `lineStackedColumnComboChart` | `employees[Age]`; Y **implicit** `CountNonNull(employee_fact[EmpID])`; Y2 `[SalaryMAX]` `[SalaryMEDIAN]` `[SalaryMIN]` | "Min, med, max salaries by headcount" |
| Table | `tableEx` | `employees[Employee_Name]`, **implicit** `Sum(employees[Age])`, `departments[Department]`, `[SalaryAVG]` | none |

Same 7 slicers. 16 shapes, 13 action buttons.

### Page 3 — HR Metrics Overview

Three identical rows (Turnover, Retention, Attrition), each: a KPI card, a
"last N years" KPI card, an area chart over time, a waterfall of period-on-period
change, and a bar chart by department.

| Visual | Type | Fields | Title |
|---|---|---|---|
| Card ×2 | `card` | `[Turnover Rate]`, `[TurnoverRateLastYears]` | "Turnover rate", "Turnover rate LYs" |
| Area | `areaChart` | `DateTable[Date].[Year]` → `[Month]` × `[Turnover Rate]` | "Turnover rate over Years" |
| Waterfall | `waterfallChart` | same axis × `[Turnover Rate vs Prev Period]` | "Turnover rate: YoY change" |
| Bar | `barChart` | `departments[Department]` → `positions[Position]`, series `employees[Sex]`, Y `[Turnover Rate]` | "Turnover rate by Department and Sex" |
| …the same five for Retention | | | "Retention rate over Years", "Retention rate: YoY change", … |
| …the same five for Attrition | | | "Attrition rate over **Yearss**", "Attrition rate: YoY change", … |
| What-if slicer | `slicer` | `Rates years  window[Rates years  window]`, numeric, **default 3** | header "Years to look back on" |

Eight slicers, 15 shapes, 7 images (the card backdrops), 5 action buttons.

### Page 4 — Employees Breakdown

A single-employee profile card, driven by a `listSlicer` on
`employees[Employee_Name]` that is hidden and revealed by the
`Show`/`Close Name selector` bookmarks.

| Visual | Type | Fields | Title |
|---|---|---|---|
| Avatar | `simpleImage` custom visual | `Photos[ImageURL]` | none |
| Card | `card` | `[SalaryMAX]` | "Salary / vs dept avg" |
| Card | `card` | `[EmployeeSalaryDiff vs DeptAVG %]` | none |
| Card | `card` | **implicit** `Max(employee_fact[PerformanceScore])` | "Performance Review" |
| Card | `card` | **implicit** `Sum(employee_fact[SpecialProjectsCount])` | "Number of Extra **Projectss**" |
| Card | `card` | **implicit** `Sum(employee_fact[Absences])` | "Number of Absences" |
| Donut gauge | `donutChart` | **implicit** `Sum(employee_fact[EmpSatisfaction])` + `[EmployeeSatisfactio]` | "Satisfaction Rate" |
| Donut gauge | `donutChart` | **implicit** `Sum(employee_fact[EngagementSurvey])` + `[EmployeeEngagement]` | "Engagement Rate" |
| Profile fields | 13 × `card` | all **implicit**: `Max(positions[Position])`, `Max(employees[EmploymentStatus])`, `Sum(employees[Age])`, `Max(employees[Sex])`, `Max(employees[MaritalDesc])`, `Max(employees[CitizenDesc])`, `Max(employees[DOB])`, `Count(employee_fact[EmpID])`, `Max(departments[Department])`, `Max(managers[ManagerName])`, `Max(employees[DateofHire])`, `Max(employees[TermReason])`, plus `[EmployeeTerminationDate]` | none; labelled by 14 adjacent textboxes |
| Peer table | `tableEx` | `DepartmentsEmployees[Employee_Name]`, `[Department]`, `employees[Position]`, implicit sums of `[EngagementRate]` `[SatisfactionRate]` `[Salary]`, `[EmployeesSalaryAVG vs DeptAVG]`, `[EmployeeSalary vs SelectedEmployee]` | none |

### Do titles match what the visuals show?

Mostly yes. The exceptions:

- **"Employees count"** (card) counts **active employees only**; the three age
  cards and "Average salary" next to it cover **all 311**. Four cards on one row,
  two populations, no label saying so. See §5.
- **"Min, med, max salaries by headcount"** — the x-axis is `employees[Age]`, not
  headcount; headcount is the column measure. The title names neither age nor the
  median/min/max line series clearly. "Salary by age, with headcount" would be
  closer.
- **"Turnover rate by Department and Sex"** (and its Retention/Attrition twins) —
  the category axis is a two-level drill `Department → Position`. The title does
  not mention that you can drill to position.
- **"Salary / vs dept avg"** labels a card whose only field is `[SalaryMAX]`;
  the "vs dept avg" half is a second, untitled card next to it.
- **"Engagement Rate" / "Satisfaction Rate"** donuts read correctly for a single
  selected employee only — they pair an implicit `Sum` with a `5 − AVERAGE`
  remainder, so with two or more employees in context the two halves are on
  different scales and the ring is meaningless.
- Typos on the canvas: **"Average saalary"**, **"ethic group"** (ethnic),
  **"Attrition rate over Yearss"**, **"Number of Extra Projectss"**. And in the
  model: measure **`EmployeeSatisfactio`**.

---

## 4. Quality

### DAX

**Implicit measures: 39 aggregations set in visuals rather than defined as
measures** — 36 of them on Employees Breakdown, 3 on Salary Distribution. The
whole profile card on page 4 is built from `Max(…)` and `Sum(…)` picked in the
field well. Two consequences: the numbers cannot be reused or renamed, and they
are silently wrong outside single-selection (`Sum(employees[Age])` is one employee's
age only while exactly one employee is selected).

**Descriptions: 0 of 39 measures have one.** Same for all 60-odd visible columns
and all 11 visible tables. For a portfolio model, where the point is to show
somebody else how you think, this is the single biggest documentation gap.

**Display folders:** good — `Employee`, `Salary`, `Turnover Rate`,
`Attrition Rate`, `Retention Rate`. Only `EngagementRate ProgressBar` sits loose
at the root.

**Calculated columns vs measures:** the two calculated columns are correctly
columns. Nothing that should be a column is a measure. The reverse problem is the
39 implicit aggregations, which should be measures.

**Specific DAX issues**

- `EmployeeTotalCurrent` uses `CALCULATE(…, employees[EmploymentStatus] = "Active")`,
  which **overwrites** any incoming filter on that column instead of intersecting
  it. Confirmed: breaking the measure down by `EmploymentStatus` returns 207 on
  every row, including "Terminated for Cause". No visual currently does that
  breakdown, so it is latent rather than visible — but a future slicer on
  employment status would silently do nothing. `KEEPFILTERS` fixes it.
- `EmployeesSalaryAVG vs DeptAVG` and `EmployeeSalary vs SelectedEmployee` use
  bare `/` instead of `DIVIDE`.
- `EmployeesCount_StartOfPeriod` / `EndOfPeriod` call
  `FILTER(employees, …)` — a whole-table filter where a column filter would do,
  and `<` rather than `<=` on the start boundary, so an employee hired on the
  first day of the period is not counted in the opening headcount.
- `TurnoverRateLastYears`, `AttritionRateLastYears`, `RetentionRateLastYears`
  reference the what-if measure as `'Rates years  window'[Rates years  window Value]`
  — a table-qualified measure reference, which BPA flags at severity 3.
- `Turnover Rate` and friends branch on `HASONEVALUE(DateTable[Date])` — a single
  *date*, not a single year or month. In a chart by year that is always false, so
  the `AVERAGEX` branch always runs. It happens to give the right per-year answer
  (one year in context ⇒ one row to average), but the grand total is then an
  unweighted mean across years. See §5.
- The three `… vs Prev Period` measures use `ISINSCOPE(DateTable[Date].[Month])`
  to switch between MoM and YoY — a neat touch, and another auto date/time
  dependency.
- `SalaryAVG` (hire-date relationship) and `EmployeeSalaryAVG` (no date
  relationship) are two measures for "average salary" that agree at the grand
  total ($69,020.68) and diverge under any date filter. Page 1 uses one, page 2
  the other.

### Model

| Check | Result |
|---|---|
| Bidirectional relationships | **3** — `employees↔employee_fact`, `employees↔Photos`, `employees↔DepartmentsEmployees` |
| Many-to-many | **1** — `employees[DepartmentID] ↔ DepartmentsEmployees[DepartmentID]`, and it is bidirectional too |
| Date table present | Yes, but **not marked as a date table** and it has only a `Date` column |
| Auto date/time | **On.** 4 hidden tables; all report time axes depend on them |
| Inactive relationships | 3, one of which (`employees[Department] → departments[Department]`) is an auto-detected leftover never activated by any measure |
| Relationship key types | 6 relationships join columns of different data types (e.g. `employee_fact[DeptID]` int64 → `departments[DeptID]` string) |
| Unused columns | **12** visible columns referenced by nothing |
| Redundant columns | `employees[Position]`, `employees[Department]` mirror the dimension columns — but see the correction in §2: the dimension side cannot be filtered from the employee side, so these are needed; BPA also flags `employees[Employee_Name]` vs `DepartmentsEmployees[Employee_Name]` |
| Roles | none |
| Hidden fact/key columns | none hidden — every foreign key and every raw ID is visible in the field list |

The 12 unused columns: `employee_fact[MarriedID]`, `[MaritalStatusID]`,
`[GenderID]`, `[EmpStatusID]`, `[PerfScoreID]`, `[FromDiversityJobFairID]`,
`[Termd]`, `[DaysLateLast30]`, `employees[State]`, `[Zip]`, `[HispanicLatino]`,
`Photos[ID]`. (`employee_fact[ManagerID]`, `managers[ManagerID]` and
`Photos[ID_upd]` are relationship keys, so they are used even though no visual
names them.)

### Tabular Editor BPA

The rule file was **not** at the path given
(`C:\Users\win10pro\project\steam-player-collector\bpa\BPARules.json` — that folder
does not exist). It was found at
**`C:\Users\win10pro\Downloads\project\bpa\BPARules.json`**: 68 rules, Microsoft's
Best Practice Rules minus three, belonging to the `CDPR_Health_Hub` project. That
is the file used here. Tabular Editor 2.29.0 portable was found at
`C:\Tools\TabularEditor2`.

```powershell
C:\Tools\TabularEditor2\TabularEditor.exe `
  "hrdashboard.SemanticModel\definition\database.tmdl" `
  -A "C:\Users\win10pro\Downloads\project\bpa\BPARules.json"
```

Read-only; it analysed the TMDL on disk and changed nothing.
**Exit code 1 — 270 violations across 29 rules.**

| Sev | Count | Rule |
|---|---|---|
| 3 | 15 | [Formatting] Do not summarize numeric columns |
| 3 | 12 | [Formatting] Provide format string for measures |
| 3 | 6 | [Error Prevention] Relationship columns should be of the same data type |
| 3 | 3 | [DAX Expressions] Measure references should be unqualified |
| 2 | 22 | [Formatting] Whole numbers should be formatted with thousands separators and no decimals |
| 2 | 15 | [Formatting] Hide foreign keys |
| 2 | 10 | [Formatting] Percentages should be formatted with thousands separators and 1 decimal |
| 2 | 8 | [Performance] Reduce usage of calculated tables |
| 2 | 6 | [DAX Expressions] Filter column values with proper syntax |
| 2 | 6 | [DAX Expressions] Use the DIVIDE function for division |
| 2 | 6 | [Performance] Minimize Power Query transformations |
| 2 | 5 | [Performance] Date/calendar tables should be marked as a date table |
| 2 | 4 | [Performance] Remove auto-date table |
| 2 | 4 | [Performance] Do not use floating point data types |
| 2 | 4 | [Performance] Set IsAvailableInMdx to false on non-attribute columns |
| 2 | 4 | [Formatting] Hide fact table columns |
| 2 | 3 | [DAX Expressions] Inactive relationships that are never activated |
| 2 | 2 | [Performance] Consider a star-schema instead of a snowflake architecture |
| 2 | 1 | [Performance] Model should have a date table |
| 2 | 1 | [Performance] Avoid excessive bi-directional or many-to-many relationships |
| 2 | 1 | [Performance] Many-to-many relationships should be single-direction |
| 2 | 1 | [Performance] Remove redundant columns in related tables |
| 2 | 1 | [Performance] Reduce number of calculated columns |
| 2 | 1 | [Maintenance] Remove unnecessary columns |
| 1 | **107** | [Maintenance] Visible objects with no description |
| 1 | 10 | [Formatting] Mark primary keys |
| 1 | 5 | [Formatting] First letter of objects must be capitalized |
| 1 | 4 | [Maintenance] Ensure tables have relationships |
| 1 | 3 | [Performance] Check if bi-directional and many-to-many relationships are valid |

Notes on reading this list: "Model should have a date table" fires because
`DateTable` is not *marked* as one — the model does have a calendar.
"Reduce usage of calculated tables" counts the four auto date/time tables plus the
measure-container, field-parameter and what-if tables, which are standard patterns
and would normally be annotated as deliberate exceptions. The four
floating-point columns are real: `employee_fact[PositionID]`, `[ManagerID]`,
`[EngagementSurvey]` and `managers[ManagerID]` are `Double` where two of them are
integer keys.

For comparison, the `CDPR_Health_Hub` project this rule file comes from sits at
**0 violations** with its deliberate exceptions annotated in TMDL. That is the
standard this project is being measured against, and the gap is 270.

---

## 5. Numbers: do the headline KPIs compute correctly?

Queried against the live model. Two sets of figures, because the report-level
filter `positions[Position] <> 'President & CEO'` applies to everything on the
canvas but not to a bare DAX query.

| KPI | Measure | Raw model | **As displayed** (report filter applied) |
|---|---|---|---|
| Employees count | `[EmployeeTotalCurrent]` | 207 | **206** |
| Average salary | `[EmployeeSalaryAVG]` | $69,020.68 | **$68,436.88** |
| Average age | `[EmployeeAgeAVG]` | 41 | **41** |
| Min age | `[EmployeeAgeMIN]` | 28 | **28** |
| Max age | `[EmployeeAgeMAX]` | 69 | **69** |

### Check 1 — headcount is correct, and it is active-only

| | |
|---|---|
| Rows in `employees` | 311 |
| Distinct `EmpID` | 311 (no duplicates) |
| Rows in `employee_fact` | 311, distinct `EmpID` 311 |
| `EmploymentStatus = "Active"` | **207** |
| `<> "Active"` | 104 |
| `SUM(employee_fact[Termd])` | **104** — agrees independently |

`[EmployeeTotalCurrent]` = 207 and an independent `COUNTROWS` with the same filter
= 207. The measure is right, and the two source tables agree on who has left.
**Verdict: correct.**

### Check 2 — the KPI row mixes two populations

| Status | Headcount | Avg salary | Avg age |
|---|---|---|---|
| Active | 207 | $70,694.03 | 40.77 |
| Voluntarily Terminated | 88 | $63,949.20 | 43.17 |
| Terminated for Cause | 16 | $75,264.88 | 40.00 |
| **All** | **311** | **$69,020.68** | **41.41** |

"Employees count" uses the Active row (207). "Average salary" and the three age
cards use the All row. Active-only average salary would be **$70,694** against the
**$69,021** on display — a **$1,673** difference, because the 88 voluntary leavers
earned about $6,700 less than the people who stayed. Active-only average age would
be **40**, not 41.

Nothing is miscalculated; each measure does what it says. But four cards sitting
in one row, two of them answering a different question from the others, is the
kind of thing that gets noticed. **Verdict: individually correct, mutually
inconsistent.**

### Check 3 — average age, and what "age" means

| | |
|---|---|
| `AVERAGE(employees[Age])` | 41.408… |
| `[EmployeeAgeAVG]` = `ROUNDDOWN(…, 0)` | **41** |
| `MAX(DateTable[Date])` | 2020-02-28 ⇒ `Age` = 2020 − `YEAR(DOB)` |
| DOB range | 1951-01-02 … 1992-08-17 |
| `[EmployeeAgeMIN]` / `MAX` | 28 / 69 — consistent with 2020 − 1992 and 2020 − 1951 |

The arithmetic is internally consistent. Two notes:

- Ages are **as of 2020**, the padded end of the date table, and are frozen there.
  Not wrong for a fixed dataset, but undocumented, and the reference year is an
  accident of `+ 365` rather than a decision.
- `ROUNDDOWN` on an average truncates: 41.41 → 41. `ROUND` would be the
  conventional choice. It happens not to change the answer here. (Changed to
  `ROUND` later — see §8.)

Age-band distribution, which confirms both the bucket logic and the empty bucket:

| Age range | All | Active | Min | Max |
|---|---|---|---|---|
| 18-24 | — | — | — | — |
| 25-34 | 83 | 58 | 28 | 34 |
| 35-44 | 128 | 90 | 35 | 44 |
| 45-54 | 73 | 46 | 45 | 54 |
| 55-64 | 17 | 9 | 55 | 62 |
| 65+ | 10 | 4 | 65 | 69 |
| **Total** | **311** | **207** | | |

Buckets sum to 311 / 207 with no nulls, and every bucket's min/max falls inside
its own label. **Verdict: correct.**

### Check 4 — the two average-salary measures agree, at the total

`[EmployeeSalaryAVG]` = `[SalaryAVG]` = **$69,020.68** at the grand total, and both
equal a plain `AVERAGE(employee_fact[Salary])`. They differ as soon as a date
filter is in play, because `[SalaryAVG]` activates the hire-date relationship. Page 1
uses the first, page 2 the second, both labelled "Average salary". **Verdict:
correct but duplicated, and the duplication is invisible to the reader.**

### Check 5 — the padded calendar puts two dead years in every trend

| | |
|---|---|
| Last hire | 2018-07-09 |
| Last termination | 2018-11-10 |
| Last performance review | 2019-02-28 |
| `DateTable` ends | **2020-02-28** (365 days past the last review) |

Turnover / retention / attrition by year:

| Year | Left | Start | End | Hired | Turnover | Retention | Attrition |
|---|---|---|---|---|---|---|---|
| 2006 | 0 | — | 1 | 1 | 0.00% | *(blank)* | 0.00% |
| 2007 | 0 | 1 | 3 | 2 | 0.00% | 100.00% | 0.00% |
| 2008 | 0 | 3 | 6 | 3 | 0.00% | 100.00% | 0.00% |
| 2009 | 0 | 6 | 13 | 7 | 0.00% | 100.00% | 0.00% |
| 2010 | 1 | 13 | 21 | 9 | 5.88% | 92.31% | 5.88% |
| 2011 | 3 | 21 | 101 | 83 | 4.92% | 85.71% | 4.92% |
| 2012 | 8 | 101 | 138 | 45 | 6.69% | 92.08% | 6.69% |
| 2013 | 13 | 138 | 169 | 44 | 8.47% | 90.58% | 7.82% |
| 2014 | 13 | 169 | 216 | 60 | 6.75% | 92.31% | 6.75% |
| 2015 | 23 | 216 | 229 | 36 | 10.34% | 89.35% | 8.09% |
| 2016 | 22 | 229 | 221 | 14 | 9.78% | 90.39% | 8.00% |
| 2017 | 8 | 221 | 219 | 6 | 3.64% | 96.38% | 2.73% |
| 2018 | 13 | 219 | 207 | 1 | 6.10% | 94.06% | 4.23% |
| **2019** | **0** | 207 | 207 | 0 | **0.00%** | **100.00%** | **0.00%** |
| **2020** | **0** | 207 | 207 | 0 | **0.00%** | **100.00%** | **0.00%** |

The per-year figures for 2010–2018 are sound: leavers sum to 104 across all years,
matching `SUM(Termd)`, and start/end headcounts chain correctly year to year
(2018 ends at 207, which is the active headcount). **The yearly rates are
correct.**

The tails are not. 2019 and 2020 exist only because of the `+ 365`, and 2006–2009
are the pre-history when the company had fewer than 15 people. Two consequences:

1. **The three trend charts and three waterfalls each carry a flat two-year tail**
   at 0% turnover / 100% retention that is an artefact of the calendar, not a
   result.
2. **The "LYs" cards are diluted.** With the slicer at its default of 3 years,
   "last 3 years" means 2018 + 2019 + 2020 — one real year and two empty ones:

   | Card | Value at default window = 3 |
   |---|---|
   | Turnover rate LYs | **2.03%** |
   | Retention rate LYs | **98.02%** |
   | Attrition rate LYs | **1.41%** |

   2018 alone was 6.10% turnover. The card reads 2.03% — roughly a third, because
   two of the three years are blank. A visitor to the published page sees 2.03%
   and has no way to know it is 6.10% over the only real year in the window.

3. **The headline cards are unweighted means across years.** `[Turnover Rate]` at
   no date filter = **4.17%**, which is the arithmetic mean of the 15 yearly
   values — including 6 structural zeros. Weighted properly (104 leavers over an
   average headcount of about 213) all-time turnover is nearer **49%** over the
   13 years, or about 6.8% a year over the 9 years that had any leavers. The 4.17%
   is not a number that answers any question a reader would ask. Same for
   `[Retention Rate]` = 94.51% and `[Attrition Rate]` = 3.67%.

**Verdict: per-year rates correct; the totals and the "last N years" cards are
materially understated by the padded calendar and the unweighted average.**

### Check 6 — the active-employee filter does not survive slicing

Breaking `[EmployeeTotalCurrent]` down by `EmploymentStatus`:

| EmploymentStatus | True headcount | `[EmployeeTotalCurrent]` |
|---|---|---|
| Active | 207 | 207 |
| Voluntarily Terminated | 88 | **207** |
| Terminated for Cause | 16 | **207** |

The `CALCULATE` filter replaces the incoming one. No visual does this breakdown
today, so nothing on the published page is wrong — but the measure is not safe to
reuse. **Verdict: latent defect.**

---

## 6. Git

**Not a git repository.** There is no `.git` directory here or in any parent;
`git rev-parse --is-inside-work-tree` returns *fatal: not a git repository*.

A `.gitignore` **is** present, dated today, with two sensible entries:

```gitignore
**/.pbi/localSettings.json
**/.pbi/cache.abf
```

So the groundwork for version control has been laid but `git init` was never run.
Worth noting that `hrdashboard.pbix` (599 KB) sits alongside the PBIP folders —
committing both means a binary and its own serialised form in the same repo, which
defeats the point of PBIP. And `hrdashboard.SemanticModel/.pbi/cache.abf` is
already ignored, which is right: that file holds the imported data.

`README.md` contains one line: `# 01-hrdashboard`.

> **Updated.** Both of these were addressed — see §8. The folder is now a git
> repository with `*.pbix` ignored, and `README.md` has been rewritten.

---

## 7. Prioritised improvements

### Quick fixes (minutes)

1. **Fix the five typos.** "Average saalary" → "Average salary"; "ethic group" →
   "ethnic group"; "Attrition rate over Yearss" → "…Years"; "Number of Extra
   Projectss" → "…Projects"; measure `EmployeeSatisfactio` →
   `EmployeeSatisfaction`. These are on a page a recruiter will look at.
2. **Trim the date table.** Change `+ 365` to end at the last real event
   (`MAX(employees[LastPerformanceReview_Date])`, or year-end 2019). This alone
   removes the flat tails from six charts and roughly triples the three "LYs"
   cards back to honest values.
3. **Label the KPI row,** or filter it consistently. Either retitle to "Active
   employees" / "Average salary (all staff)" / "Average age (all staff)", or wrap
   the salary and age measures in the same active filter. Pick one population.
4. **Rename the two case-duplicate bookmarks** (`years`/`Years`,
   `positions`/`Positions`) so the bookmark pane is readable.
5. **Add format strings** to the 12 measures without one, and set `summarizeBy:
   none` on the 15 numeric columns BPA flags — both are property edits and both
   are severity 3.
6. **`git init`** and make a first commit. The `.gitignore` is already written.
   Decide whether `hrdashboard.pbix` belongs in the repo at all.

### Worth doing (an hour or two)

7. **Parameterise the source folder.** Add a `SourceFolder` Power Query parameter
   defaulting to the project folder and rewrite the six `File.Contents` calls
   against it. Until this is done nobody can open and refresh this project —
   which for a portfolio piece someone might clone is the most expensive single
   defect in the file.
8. **Give the date table real columns and mark it as one.** Add `Year`,
   `Quarter`, `Month No`, `Month Name` (sorted), `Year Month`; mark the table as
   a date table on `Date`; repoint every axis and every `.[Year]` / `.[Month]`
   reference in the measures off the auto hierarchy; then **turn off auto
   date/time** in options. That clears 13 BPA violations and four hidden tables,
   and makes the model's own date table the thing the report actually runs on.
9. **Replace the 39 implicit aggregations with named measures** —
   `[Performance Score]`, `[Special Projects]`, `[Absences]`, `[Employee Age]`,
   `[Satisfaction]`, `[Engagement]` and so on. Fix the two donut gauges while
   you are there: pair `AVERAGE` with `5 − AVERAGE`, not `SUM` with
   `5 − AVERAGE`, so they survive multi-selection.
10. **Write descriptions** for the 39 measures and the main columns. This is the
    107-violation rule and the clearest signal of modelling discipline a reviewer
    can see without opening a single DAX expression.
11. **Hide the plumbing.** 15 foreign keys and the four raw ID columns in
    `employee_fact` are visible in the field list; the 12 genuinely unused
    columns can be removed outright. A field list a reader can scan is worth more
    than it looks.
12. **Make `[EmployeeTotalCurrent]` filter-safe** with `KEEPFILTERS`, switch the
    two bare divisions to `DIVIDE`, unqualify the three what-if measure
    references, and change the two `FILTER(employees, …)` calls to column filters.
13. **Comment the Power Query magic numbers.** Add step descriptions saying why
    `ManagerID` null becomes 39, why 30 becomes 22, why position 17 becomes 18,
    why `Software Engineer` becomes 24, and why `Photos` skips 11 rows. Also
    delete the no-op `Table.SelectRows(…, each true)` step.
14. **Expand `README.md`** into something a visitor can read: what the dataset is,
    where it came from, that the data is fictional, what the four pages answer,
    how turnover/retention/attrition are defined, and the fact that ages are as of
    2019/2020. One page.

### Bigger changes

15. **Decide what the rate measures mean at the total, and implement that.**
    Today the headline cards are unweighted means across calendar years, including
    years with no activity. Either make the total a properly weighted
    leavers ÷ average-headcount figure, or restrict the cards to a stated period
    and say so on the canvas. This is the one place where the dashboard shows a
    number that is defensible as arithmetic but not as an answer.
16. **Flatten to a real star schema.** `employees` and `employee_fact` share a
    grain and a key; merge them into one `Employee` dimension plus, if you want a
    fact, a thin `Employment Event` table keyed on date. Then drop the three
    bidirectional relationships and the many-to-many. One table holding both the
    employee and their job title and department is also what makes the flattened
    `employees[Position]` / `[Department]` columns stop being a special case —
    see the correction in §2; do **not** simply delete them while the two-table
    shape remains. This clears most of the remaining Performance category and
    removes the snowflake finding.
17. **Replace `DepartmentsEmployees` with measures.** It is a calculated
    `SUMMARIZE` of `employees` reached through a bidirectional many-to-many,
    existing only so that three averages can be read as columns. `ALLEXCEPT`
    measures over the base tables do the same job without the extra table or the
    relationship.
18. **Replace the 108 hand-set `NoFilter` overrides on HR Metrics.** Nine
    independent charts in one container, or three small multiples, or a
    calculation group over the rate measures — anything that does not need an
    all-pairs matrix maintained by hand.
19. **Self-host the avatars.** 311 external URLs on `xsgames.co` are a live
    dependency on a free third-party service for a page that is published to the
    open web. Base64 the images into the model, or host them somewhere you
    control.
20. **Add RLS and a second published view.** An HR model is the canonical RLS
    demo — manager sees their own team. It is incompatible with Publish to web, so
    it would mean a second artefact, but "here is the same model with RLS" is a
    strong thing for a portfolio to be able to show.
21. **Adopt the BPA gate from the other project.** Commit the rule file, annotate
    the deliberate exceptions with `BestPracticeAnalyzer_IgnoreRules` in TMDL, and
    run Tabular Editor in CI the way `CDPR_Health_Hub` does. Getting from 270 to 0
    with documented exceptions is itself a portfolio artefact.

---

## 8. Changes applied

Applied 2026-11-12, in one pass, against a full backup of the folder
(`../hrdashboard-portfolio-backup-20261112`, 292 files, verified byte-identical
before any edit). The model was edited as TMDL on disk with Power BI Desktop
closed, then reopened and refreshed to verify; the report layer was edited as
JSON in the same window.

Everything below was checked against the live model after the refresh, not
predicted. Where a number changed, the old and new values are both given.

### Data sources and Power Query

- **`SourceFolder` parameter added** (text, defaults to this project folder) and
  all six queries rewritten to build their paths from it. No reference to
  `C:\Users\Admin\Downloads\hrdashboard-main\` remains anywhere in the model.
  Refreshing a clone is now one parameter change.
- **Removed the no-op step** `Table.SelectRows(…, each true)` from
  `employee_fact`.
- **Commented every magic-number repair** in the M, with the cause rather than a
  restatement. All five turned out to be explainable from the data:

  | Step | Why |
  |---|---|
  | `ManagerID` null to 39 | `managers.csv` lists *Webster Butler* twice, as 39 and with a blank ID. The 8 fact rows with no manager are all Production Technicians in his department. |
  | `ManagerID` 30 to 22 | *Michael Albert* appears as both 22 and 30; one employee points at 30, twenty-one at 22. |
  | `PositionID` 17 to 18 | Both are titled *Production Manager*. |
  | `Software Engineer` to 24 | ID 23 covers two unrelated titles, *Shared Services Manager* and *Software Engineer*, while 24 is already *Software Engineer*. |
  | `Photos` skip 11 | The workbook has 322 data rows of which the last 11 are completely empty; sorting by ID ascending puts those nulls first. 311 avatars remain, IDs 1 to 311, no real rows lost. |

  The same applies to the `departments` remap: DeptIDs 5 and 6 are both
  *Production*, 1 and 4 are both *Software Engineering*.

  These are **M comments, not Power Query step descriptions.** A TMDL partition
  stores only the expression, so a description set in the Properties pane does
  not round-trip to disk. The comments survive a save, but editing a step in the
  UI can drop them.

- **Cast five relationship keys back to `Int64`** (`departments[DeptID]`,
  `positions[PositionID]`, `managers[ManagerID]`, `employee_fact[PositionID]`,
  `[ManagerID]`). `Table.AddColumn` and `Table.ReplaceValue` had left them as
  text or decimal while their counterparts were whole numbers. *Beyond the
  brief*, but it cleared the last severity-3 BPA finding and three
  floating-point ones.

### Date table

- **Rebuilt** as `ADDCOLUMNS(CALENDAR(DATE(2006,1,1), DATE(2018,12,31)), …)` with
  `Year`, `Quarter`, `Month No`, `Month Name` (sorted by `Month No`) and
  `Year Month`. 4,748 rows, verified.
- **Marked as the date table** (`dataCategory: Time`, `isKey` and an explicit
  `dataType: dateTime` on `Date` — BPA requires all three before it accepts a
  table as a calendar).
- **Auto date/time switched off** (`__PBI_TimeIntelligenceEnabled = 0`) and all
  four hidden auto tables deleted, with their three variation relationships.
- **Every `.[Year]` and `.[Month]` reference repointed** to the real columns — in
  9 measures, 8 visuals, the report-level filter and all 14 bookmarks. The month
  axis uses `Month Name`, and `ISINSCOPE` in the three *vs Prev Period* measures
  was moved to match, so the month-on-month branch still fires (confirmed: 83
  non-zero monthly deltas).
- **Dropped the never-activated relationship** `employees[Department]` to
  `departments[Department]`. Twelve relationships became eight.

The two dead years are gone. 2019 and 2020 existed only because the old calendar
ran 365 days past the last performance review; they contributed a flat tail to
six charts and diluted every "last N years" card.

### Age

- `employees[Age]` now computes against an explicit reference date of
  **31 December 2018** instead of `MAX` of the calendar, documented in the column
  description. Because the reference date is 31 December everyone has had their
  birthday, so the year difference is the exact age.
- Every age dropped by exactly 2, the old reference year having been 2020.
- The empty `"18-24"` band is renamed **`"Under 25"`**. It stays empty: the
  youngest employee is now 26.

### Overview KPI row

`EmployeeSalaryAVG` and the three age measures now carry the same
`KEEPFILTERS(EmploymentStatus = "Active")` as the headcount, and the cards are
retitled. Values as the canvas shows them, with the report-level CEO filter
applied:

| Card | Was | Now |
|---|---|---|
| Employees count to **Active employees** | 206 | 206 |
| Average salary to **Average salary (active)** | $68,437 | **$69,824** |
| Min age of employees to **Min age (active)** | 28 | **26** |
| Average age of employees to **Average age (active)** | 41 | **39** |
| Max age of employees to **Max age (active)** | 69 | **67** |

Two knock-on effects, both intended:

- The Overview matrix and the sex-by-ethnicity combo chart also read
  `EmployeeSalaryAVG`, so they now show active-only salary alongside their
  active-only headcounts. That is the consistency the change was for.
- `EmployeeSalary vs SelectedEmployee` was repointed to a new
  **`EmployeeSalaryAVGAllStaff`** (`AVERAGE(employee_fact[Salary])`, unfiltered)
  so the peer comparison is numerically unchanged. This was necessary, not
  cautious: the active-only measure is blank for a terminated employee, which
  would have broken the comparison for 104 of the 311 people.

### Turnover, attrition and retention

Replaced the unweighted average across years with a pooled rate: summed
numerator over summed denominator, so each year is weighted by its headcount and
a single year still returns exactly that year's rate.

```dax
Turnover Rate =
VAR YearsInScope = VALUES(DateTable[Year])
VAR Leavers = SUMX(YearsInScope, [EmployeesCount_Left])
VAR AverageHeadcount = SUMX(YearsInScope, [EmployeesCount_AvgPerPeriod])
RETURN DIVIDE(Leavers, AverageHeadcount)
```

**Per-year values are unchanged for all 13 years** — verified against the old
measures before the change and against the live model after it.

| | Was | Now |
|---|---|---|
| Turnover rate (total) | 4.17% | **7.22%** |
| Attrition rate (total) | 3.67% | **6.11%** |
| Retention rate (total) | 94.51% | **92.22%** |
| Turnover rate LYs (window 3) | 2.03% | **6.53%** |
| Attrition rate LYs | 1.41% | **5.02%** |
| Retention rate LYs | 98.02% | **93.57%** |

Both effects are in play: pooling changes the weighting, and trimming the
calendar removes two empty years. The LYs cards at the default window of 3 now
cover 2016 to 2018, three real years instead of one real plus two blank. The
denominator for the total is 104 leavers over 1,440.5 summed average headcount.

### DAX corrections

- **`KEEPFILTERS` in `EmployeeTotalCurrent`.** Broken down by employment status
  it used to return 207 on all three rows; it now returns 207 / blank / blank. No
  visual did that breakdown, so nothing on the page was wrong — but the measure
  is now safe to reuse.
- **`DIVIDE` instead of `/`** in `EmployeesSalaryAVG vs DeptAVG`,
  `EmployeeSalary vs SelectedEmployee`, `EmployeeSalaryDiff vs DeptAVG %` and the
  SVG progress bar.
- **What-if measure reference unqualified** in the three `…RateLastYears`
  measures.
- **Column filters instead of `FILTER(employees, …)`** in
  `EmployeesCount_StartOfPeriod` and `EndOfPeriod` and `EmployeesHiredPerPeriod`.
  Verified to return identical values.
- **One headcount definition** for both ends of the period — see below.
- **`ROUND` instead of `ROUNDDOWN`** in `EmployeeAgeAVG`. The raw active average
  is 38.65, so the card reads **39** rather than truncating to 38.
- **`USERELATIONSHIP` arguments reordered** to `FromColumn, ToColumn` in nine
  measures. Identical in DAX, but the BPA rule's regex is order-sensitive and was
  otherwise reporting both date relationships as never activated.

### One headcount definition

The opening and closing headcounts originally disagreed about their boundaries.
`StartOfPeriod` used `hire < D` and `termination >= D`; `EndOfPeriod` used the
same, so someone hired on the closing day was never counted as present. Both now
apply a single test:

> **Employed on date D** = hired on or before D, and not terminated on or
> before D.

```dax
EmployeesCount_EndOfPeriod =
VAR PeriodEnd = MAX(DateTable[Date])
RETURN
    CALCULATE(
        DISTINCTCOUNT(employees[EmpID]),
        employees[DateofHire] <= PeriodEnd,
        ISBLANK(employees[DateofTermination]) || employees[DateofTermination] > PeriodEnd
    )
```

**Nothing changes at the yearly grain.** Opening and closing headcount, turnover,
retention and attrition are identical for all 13 years, maximum movement
**0.00 pp**, because nobody in the data is hired or terminated on 1 January or
31 December. The totals are also untouched: turnover 7.22%, attrition 6.11%,
retention 92.22%, and the LYs cards at window 3 stay at 6.53% / 5.02% / 93.57%.

**Monthly figures do move**, in 14 of the 156 months, because month boundaries
are hit often: 6 people were terminated on the 1st of a month, 17 hired on a
month-end and 4 terminated on a month-end. Every one of those moves is the same
double-count being undone.

The clearest case is **September 2013**, the largest movement at 5.84 pp. Nine
people were hired on 30 September 2013:

| | Was | Now |
|---|---|---|
| Opening headcount | 154 | 154 |
| Closing headcount | 153 | **162** |
| Hired during the month | 9 | 9 |
| Retention `(closing − hired) ÷ opening` | `(153 − 9) ÷ 154` = **93.51%** | `(162 − 9) ÷ 154` = **99.35%** |

The old closing headcount excluded those nine because `hire < 30 September` was
false, yet `EmployeesHiredPerPeriod` still counted them as hires and subtracted
them. They were penalised twice: absent from the closing count *and* deducted
from it. Monthly retention was understated as a result, and the new definition
removes the contradiction rather than introducing one.

### Implicit measures

All **39 implicit aggregations are gone** from the report (0 remain, including
inside bookmark state). **23 new measures** were added in display folders
*Employee profile*, *Engagement* and *Peer comparison*, each with a description.

Two bugs surfaced doing this:

- **The "Employee ID" card was showing `1`.** It was bound to
  `Count(employee_fact[EmpID])`, which is 1 for a single selected employee. It
  now shows the payroll number (10081 for the default selection).
- **The satisfaction and engagement rings paired `SUM` with `5 − AVERAGE`.** They
  now pair `AVERAGE` with `5 − AVERAGE`, so the two segments total exactly 5 both
  for one employee and for the whole company. Previously the ring was only
  coherent for a single selection.

`EmployeeSatisfactio` renamed to **`EmployeeSatisfaction`**, all six references
updated. Its formula is unchanged as instructed, but note the name describes the
*gap* to 5 rather than satisfaction; the new `SatisfactionScore` is the actual
average. Both descriptions say so.

### The profile-card bug

Verifying the new measures turned up a defect that was live on the published
page. **Position, Department and Manager showed the same three values for every
employee.**

They were bound to `Max(positions[Position])`, `Max(departments[Department])` and
`Max(managers[ManagerName])`. Selecting an employee filters `employee_fact`
through the bidirectional relationship, but it does **not** filter those three
dimensions: they sit on the *one* side of their relationships to the fact, and
filters travel one-side-to-many, not the reverse. So each card returned the
alphabetically last value across the whole dimension, regardless of selection.

| Card | Showed, for everyone | Truth for *Boutwell, Bonalyn* |
|---|---|---|
| Position | Sr. Network Engineer | **Sr. Accountant** |
| Department | Software Engineering | **Admin Offices** |
| Manager | Webster Butler | **Brandon R. LeBlanc** |

Fixed by reading position and department from the columns Power Query already
flattens onto the employee row, and the manager by key:

```dax
EmployeePosition   = SELECTEDVALUE(employees[Position])
EmployeeDepartment = SELECTEDVALUE(employees[Department])
EmployeeManager    = LOOKUPVALUE(managers[ManagerName],
                         managers[ManagerID], SELECTEDVALUE(employee_fact[ManagerID]))
```

Verified across four employees, each resolving to their own correct values. This
is why §2's "redundant duplicates" framing was wrong and has been corrected: the
flattened columns are the only path from a selected employee to their job title
and department while the model keeps its two-table shape.

### Housekeeping

- **Format strings on every measure.** Whole numbers `#,0`, averages `#,0.0`,
  percentages `#,0.0%;-#,0.0%;#,0.0%`, currency unchanged. Text-returning
  measures carry an annotation instead; see below.
- **`summarizeBy: none`** on all numeric attribute columns.
- **Foreign keys and raw IDs hidden**: 15 key columns plus `Salary`,
  `EngagementSurvey`, `SpecialProjectsCount`, `Absences`.
  `employee_fact[EmpSatisfaction]` stays visible because it is a category axis.
- **`isAvailableInMdx: false`** on the 21 columns that hiding had just made
  non-attribute. *Beyond the brief*, but hiding them is what made the rule fire,
  so this finishes the job.
- **12 unused columns deleted**: `employees[State]`, `[Zip]`,
  `[HispanicLatino]`; `employee_fact[MarriedID]`, `[MaritalStatusID]`,
  `[GenderID]`, `[EmpStatusID]`, `[PerfScoreID]`, `[FromDiversityJobFairID]`,
  `[Termd]`, `[DaysLateLast30]`; `Photos[ID]`. Each is also removed in Power
  Query rather than loaded and hidden.
- **Descriptions on all 62 measures**, all 11 tables and every visible column.
- `Photos[ImageURL]` given the `ImageUrl` data category.

### Report

- **Typos fixed**: "Average saalary", "ethic group", "Attrition rate over
  Yearss", "Number of Extra Projectss".
- **Bookmarks renamed**: `years` to *Overview - headcount by year*, `Years` to
  *Salary - distribution by year*, `positions` to *Overview - count by position*.
  Page ownership confirmed from each bookmark's target visuals.
- **"Fictional training data" note** added to all four pages in `#605e5c`, the
  grey the pages already use for field labels, placed above the KPI band clear of
  existing visuals.
- The peer-table advanced filter converted from `Sum(EngagementRate) is not
  blank` to `[PeerEngagementRate] is not blank`, matching the measure-filter
  shape the Overview matrix already used.
- Bookmark cached state was updated too. All 14 bookmarks set
  `suppressData: true`, so those copies were inert — but leaving them would mean
  the file still referred to aggregations the visuals no longer use.
- One stale `queryRef` corrected: a card read `_DAX_measures.EmployeeGeMIN` while
  its field bound to `EmployeeAgeMIN`.

All 250 report JSON files parse. Zero references remain to `LocalDateTable`,
`PropertyVariationSource`, `Date Hierarchy`, any deleted column, or any implicit
aggregation.

### Best Practice Analyzer

Same rule file as `CDPR_Health_Hub` (68 rules), found at
`C:\Users\win10pro\Downloads\project\bpa\BPARules.json` — not the path in the
original brief, which does not exist. Tabular Editor 2.29.0 from
`C:\Tools\TabularEditor2`.

| | Violations | Severity 3 | Exit code |
|---|---|---|---|
| Before | 270 | 36 | 1 |
| After | **20** | **0** | **0** |

Re-run after the Desktop save to confirm nothing regressed: still 20, exit 0.

**On how the 250 were cleared.** Most were fixed by changing the model. Some are
suppressed by a `BestPracticeAnalyzer_IgnoreRules` annotation, the same mechanism
`CDPR_Health_Hub` documents. There are **23 annotated objects covering 38
rule-instances**; the majority sit on measures created in this pass, where they
prevented new findings rather than clearing old ones. Taking the most
conservative reading — counting every annotation as "not repaired" — the score is
20 remaining plus 38 annotated, against 270 at the start.

Annotated, with the reason:

| Object | Rule | Why |
|---|---|---|
| `_DAX_measures` | ENSURE_TABLES_HAVE_RELATIONSHIPS | A measure container holds no data and needs no relationship. |
| `DateTable`, `DepartmentsEmployees`, `Rates years  window`, `_selectorPage1` | REDUCE_USAGE_OF_CALCULATED_TABLES | The rule's expression is literally `1=1`, so it flags every calculated table. A DAX calendar, a bridge, a what-if parameter and a field parameter are all standard patterns. |
| `Rates years  window`, `_selectorPage1` | ENSURE_TABLES_HAVE_RELATIONSHIPS | Parameter tables are read by DAX, not through a relationship. |
| `_selectorPage1[_selectorPage1 Fields]` | UNNECESSARY_COLUMNS | The field parameter's `NAMEOF` payload; it cannot be deleted. |
| `employee_fact[EmpSatisfaction]` | HIDE_FACT_TABLE_COLUMNS | A 1 to 5 rating used as a category axis, so it stays visible. |
| 13 text measures | PROVIDE_FORMAT_STRING_FOR_MEASURES, INTEGER_FORMATTING | They return text. Both rules test `FormatString` with no data-type check, so they flag every text measure. |
| `EmployeeID` | INTEGER_FORMATTING | An identifier, not a quantity; `#,0` would render 10081 as "10,081". |
| `EmployeeDOB`, `EmployeeHireDate` | INTEGER_FORMATTING | They carry a date format string, which the rule reads as a non-numeric format. |

**The 20 that remain**, all structural or stylistic, and all already in §7's
"bigger changes":

| Count | Rule | Note |
|---|---|---|
| 6 | Minimize Power Query transformations | Fires on `Table.AddColumn`, `Sort`, `Distinct`, `NestedJoin`. Unavoidable without cleaning the data upstream. |
| 5 | First letter of objects must be capitalized | The five lowercase table names. Renaming means repointing every visual binding. |
| 3 | Check if bi-directional and many-to-many relationships are valid | The three bidirectional relationships — a prompt to review, not a defect. |
| 2 | Remove redundant columns in related tables | `employees[Employee_Name]` and `[Department]`, both artefacts of the two-table shape. |
| 1 | Many-to-many should be single-direction | The `DepartmentsEmployees` bridge. |
| 1 | Avoid excessive bi-directional or many-to-many relationships | Model level, same cause. |
| 1 | Consider a star-schema instead of a snowflake | `employee_fact` sharing a grain with `employees`. |
| 1 | Do not use floating point data types | `employee_fact[EngagementSurvey]` is a genuine fractional score. Converting to `Decimal` changes stored precision, which is a judgement call left alone. |

### Not done, and why

- **`Photos` `Table.Skip(11)`.** Documented rather than replaced. Filtering on
  `ID <> null` would say the same thing and survive the workbook gaining rows.
- **Table renaming, the star-schema collapse, the 108 interaction overrides, RLS
  and self-hosted avatars.** All still as written in §7.

### Version control

`git init`, with `*.pbix` added to the existing `.gitignore` — the PBIP folders
are the source of truth, and committing the binary alongside its own serialised
form defeats the point. 289 files tracked. No remote.

Identity is set **repo-local**, not global, to
`Iryna-Petruk <125769579+Iryna-Petruk@users.noreply.github.com>`. The first
commit was amended onto it and the pre-amend object purged with
`git reflog expire --expire=now --all` followed by `git gc --prune=now`, because
`--amend` leaves the old commit — and the address it carried — reachable in the
reflog. A scan of every remaining commit object confirms none contains the
previous address.

Two commits: the project as reviewed and fixed, then the single-headcount
definition and the `ROUND` change together.
