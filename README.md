# HR Overview Dashboard

A four-page Power BI dashboard over a fictional HR dataset: headcount and
demographics, salary distribution, turnover and retention, and a per-employee
profile. Saved as a Power BI Project (PBIP), so the semantic model and the
report are both plain text on disk.

**The data is fictional training data.** It is a well-known public HR practice
dataset of 311 invented employees with invented salaries, review scores and
termination reasons, and the portrait images are generated avatars. Nothing here
describes a real company or a real person.

## Contents

| Path | What it is |
|---|---|
| `hrdashboard.pbip` | The project file to open in Power BI Desktop |
| `hrdashboard.SemanticModel/` | The model as TMDL: tables, relationships, measures, Power Query |
| `hrdashboard.Report/` | The report as JSON: pages, visuals, bookmarks, theme |
| `employees.csv` | One row per employee: name, demographics, hire and termination dates |
| `employee_fact.csv` | One row per employee: salary, performance, engagement, attendance |
| `departments.csv`, `managers.csv`, `positions.csv` | Dimension lookups |
| `people_image_urls_and_ids.xlsx` | Avatar URLs, one per employee |
| `REVIEW.md` | A code-review style assessment of the project and what was changed |

## Run it locally

Every query builds its file path from a single Power Query parameter,
**`SourceFolder`**, which holds the folder containing the five CSVs and the
workbook. It is committed as a placeholder:

```
C:\path\to\hr-overview-dashboard
```

That is deliberately not a real path on anyone's machine. Power Query cannot
resolve a path relative to the project, so the alternative would be committing
an absolute path — which means committing whoever's user name happens to be in
it. After cloning, point it at your own copy:

1. Open `hrdashboard.pbip` in Power BI Desktop.
2. **Home → Transform data → Edit parameters**.
3. Set `SourceFolder` to the full path of your clone, with no trailing
   backslash, for example `C:\projects\hr-overview-dashboard`.
4. **Home → Refresh**.

That is the only change needed. Nothing else in the model hard-codes a path.

Power BI writes the value you type into
`hrdashboard.SemanticModel/definition/expressions.tmdl`, so it shows up as a
modified file in `git status`. Either leave it uncommitted, or restore it before
committing:

```bash
git restore "hrdashboard.SemanticModel/definition/expressions.tmdl"
```

To keep `git status` clean permanently, clone to a fixed short path and create a
directory junction so the committed placeholder resolves on its own — no
elevation needed, and nothing is copied:

```bat
mklink /J C:\path\to\hr-overview-dashboard "<path to your clone>"
```

Auto date/time is switched off for this file, so if you add date columns you get
no hidden per-column calendars; use the `DateTable` below instead.

## The four pages

Only **Overview** appears in the page tabs. The other three are hidden in view
mode and reached through the button rail on the left of every page, so the
report navigates like an app rather than a stack of tabs.

**Overview** — who works here now. Active headcount, average salary and the age
range of the current workforce, then breakdowns by sex, ethnicity, marital
status, department and position, and a recruitment-source matrix. A selector
swaps the lower breakdown between age band and hire source.

**Salary Distribution** — what people are paid. Min, 25th percentile, median,
average, 75th percentile and max as cards, a box-plot-style column chart of the
distribution by year (switchable to performance rating), salary against
satisfaction split by employment status, and salary against age with headcount.

**HR Metrics Overview** — are people staying. Turnover, retention and attrition,
each as a current rate, a rate over the last N years, a trend over time, a
period-on-period waterfall, and a breakdown by department and sex. The *Years to
look back on* slicer sets N and defaults to 3.

**Employees Breakdown** — one person at a time. Photo, job and status, salary
against the department average, performance rating, engagement and satisfaction
rings, personal and company details, and a table of their department colleagues
for comparison. Pick a person with the name selector button at the top.

## How the rates are defined

All three are **pooled** across whatever years are in view: the totals are not an
average of yearly percentages but a single ratio built from summed numerators and
denominators, so each year is weighted by how many people worked there. Looking at
one year gives exactly that year's rate.

Let, for each year in scope:

- **leavers** — distinct employees whose termination date falls in the year
- **voluntary leavers** — the same, restricted to `Voluntarily Terminated`
- **opening headcount** — hired on or before 1 January and not yet left
- **closing headcount** — hired before 31 December and not yet left
- **average headcount** — the mean of opening and closing
- **hires** — distinct employees whose hire date falls in the year

Then:

```
Turnover rate  = sum(leavers)            / sum(average headcount)
Attrition rate = sum(voluntary leavers)  / sum(average headcount)
Retention rate = sum(closing - hires)    / sum(opening headcount)
```

Turnover counts everyone who left, for any reason. Attrition counts only
resignations, so it is always the lower of the two. Retention asks how much of
the opening headcount was still there at the end, ignoring anyone hired during
the period.

Over the whole 2006–2018 span this gives turnover **7.2%**, attrition **6.1%**
and retention **92.2%**, from 104 leavers of whom 88 resigned.

## The age reference date

`employees[Age]` is age in whole years at a fixed reference date of
**31 December 2018**, the end of the last year that has any hires or
terminations. It is computed when the model refreshes and does not move as real
time passes, which is what you want for a dataset that stops in 2019.

Because the reference date is 31 December, everyone has already had their
birthday that year, so `2018 - YEAR(DOB)` is the exact age rather than an
approximation. Ages run from 26 to 67; the `Under 25` band exists for
completeness and is empty.

To re-base the ages, change the `ReferenceDate` variable in the `Age` calculated
column and refresh.

## Model shape

Eleven tables. `employees` and `employee_fact` are both one row per person and
are joined one-to-one; `departments`, `managers` and `positions` are lookups
hanging off `employee_fact`.

`DateTable` is a DAX calendar covering 1 January 2006 to 31 December 2018 with
`Year`, `Quarter`, `Month No`, `Month Name` and `Year Month`, marked as the
model's date table. It has **no active relationship** to `employees`: the hire
date and termination date relationships are both inactive, and each measure
switches on the one it needs with `USERELATIONSHIP`. That is what lets the same
calendar drive both "hired in 2015" and "left in 2015" without two date tables.

`DepartmentsEmployees` is a calculated bridge that backs the peer comparison
table. `_DAX_measures` holds all 62 measures and no data. `_selectorPage1` and
`Rates years  window` are a field parameter and a what-if parameter.

Measures are grouped into display folders: Employee, Employee profile, Salary,
Engagement, Peer comparison, Turnover Rate, Attrition Rate, Retention Rate.
Every one carries a description, so hovering a field in the field list explains
what it measures.

## Checking the model

The model is kept clean against Microsoft's Best Practice Rules for tabular
models. With Tabular Editor 2 installed, from this folder:

```powershell
& "C:\Tools\TabularEditor2\TabularEditor.exe" "hrdashboard.SemanticModel\definition\database.tmdl" -A "path\to\BPARules.json"
```

It is read-only and changes nothing. Exit code 0 means no severity-3 findings.
The violations that remain are deliberate and are listed in `REVIEW.md`, along
with the objects annotated as exceptions.

## Notes

- There is no row-level security. An HR model is the textbook case for it, but
  RLS is incompatible with Publish to web, which is how this report is shared.
- The avatar images are fetched from a third-party host at render time, so the
  profile page needs an internet connection and depends on that host staying up.
- Salary is in US dollars and is not inflation-adjusted, so comparing 2006 with
  2018 compares nominal figures.
