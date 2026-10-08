# Power BI Build Guide

Builds a 4-page Power BI report on top of `FPA_Budget_vs_Actual_Forecast_Model.xlsx`. Allow about 1.5 to 2 hours. Power BI Desktop is Windows-only and free.

Files in this folder:

| File | Use |
|---|---|
| `measures.dax` | All 44 measures, pasted in one step |
| `Larkspur_FPA_theme.json` | Report colors that match the Excel dashboard |
| `csv/` | The same 7 tables as CSV, as a fallback data source |

## 1. Load the data

1. Open the workbook in Excel once and save it. Power BI reads the stored values, so always save in Excel before refreshing Power BI.
2. Power BI Desktop: **Home > Get data > Excel workbook**, pick the workbook.
3. In the Navigator, tick these seven **tables** (table icon, not the sheet icon): `fact_PnL`, `fact_Units`, `dim_Account`, `dim_Product`, `dim_Scenario`, `dim_Date`, `tbl_Commentary`.
4. Click **Transform data** and confirm types: `Date` = Date, `Amount` and `Units` = Decimal/Whole number, `AccountKey`, `Sign`, `IsClosed`, sort columns = Whole number. **Close & Apply**.

## 2. Build the star schema (Model view)

Create these relationships, all many-to-one, single direction, from the fact side to the dimension:

| From (many) | To (one) |
|---|---|
| `fact_PnL[Date]` | `dim_Date[Date]` |
| `fact_PnL[Scenario]` | `dim_Scenario[Scenario]` |
| `fact_PnL[AccountKey]` | `dim_Account[AccountKey]` |
| `fact_PnL[Product]` | `dim_Product[Product]` |
| `fact_Units[Date]` | `dim_Date[Date]` |
| `fact_Units[Scenario]` | `dim_Scenario[Scenario]` |
| `fact_Units[Product]` | `dim_Product[Product]` |
| `tbl_Commentary[AccountKey]` | `dim_Account[AccountKey]` |

Power BI may auto-detect some of them. Delete any it creates that are not in this list.

Then:

- Right-click `dim_Date` > **Mark as date table** > `Date`.
- Sort by column: `dim_Date[Month]` and `dim_Date[MonthYear]` by `MonthNum`; `dim_Account[CategoryName]` by `CategorySort`; `dim_Account[Account]` by `AccountKey`; `dim_Product[Product]` by `ProductSort`; `dim_Scenario[Scenario]` by `ScenarioSort`.
- Hide the key and sort columns on the fact tables from report view.

## 3. Add the measures

1. **Home > Enter data**, leave the grid empty, name the table `_Measures`, Load.
2. Open **DAX query view** (left rail), paste the full contents of `measures.dax`.
3. Click **Update model with changes**. All measures are added to `_Measures`.
4. Click **Run** to execute the validation query at the bottom and compare with section 6.
5. Set formats in Model view: currency, 0 decimals for `$` measures; percentage, 1 decimal for `%` and `pts` measures; whole number for units; currency, 2 decimals for the two price measures.

How the key measures work, which is worth being able to explain:

- **`Budget`** is restricted to closed months with `dim_Date[IsClosed] = 1`, so year-to-date Actual is never compared against a 12-month budget.
- **`Variance $`** multiplies `Actual - Budget` by `dim_Account[Sign]` (+1 revenue, -1 cost) account by account. Favorable is positive on every line, and the total over all accounts equals the EBITDA variance.
- **`Volume Variance`** and **`Price Variance`** iterate over products: (actual units - budget units) x budget price, and (actual price - budget price) x actual units. Together they equal the revenue variance.
- **Forecast** is a 9+3 scenario: it already contains actuals for closed months, so `Forecast FY` is a full-year number.

## 4. Apply the theme

**View > Themes > Browse for themes**, pick `Larkspur_FPA_theme.json`. Blue = actual/forecast, gray = budget, green/red = favorable/unfavorable only.

## 5. Report pages

Put one **Month** slicer (`dim_Date[MonthYear]`, style: Between or dropdown) on pages 1 to 3 and sync it (View > Sync slicers). With nothing selected, every page shows year-to-date through the last closed month.

### Page 1: Executive Summary

| Visual | Fields |
|---|---|
| 4 cards (new card visual, with reference label) | `Revenue Actual` (ref: `Revenue Var %`), `Gross Margin % Actual` (ref: `Gross Margin Var pts`), `EBITDA Actual` (ref: `EBITDA Var %`), `EBITDA Margin % Actual` |
| Clustered column: revenue by month | X `dim_Date[MonthYear]`, Y `Revenue`, Legend `dim_Scenario[Scenario]`; visual filter Scenario = Budget, Forecast |
| Line: EBITDA by month | X `dim_Date[MonthYear]`, Y `EBITDA`, Legend `dim_Scenario[Scenario]`; visual filter Scenario = Budget, Forecast. Add an X-axis constant line after Sep-26 labelled "Forecast" |
| Waterfall: EBITDA variance walk | Category `dim_Account[CategoryName]` then `dim_Account[Account]` (drill-down), Y `Variance $`. Sentiment colors: increase green, decrease red |
| Text box | Two or three takeaways from the Commentary tab |

Turn off the Month slicer's effect on the two trend charts (Format > Edit interactions) so they always show 12 months.

### Page 2: P&L Budget vs. Actual

- **Matrix**: Rows `dim_Account[CategoryName]` > `dim_Account[Account]`; Values `Budget`, `Actual`, `Variance $`, `Variance %`, `Variance Status`.
- Turn **off the grand total row** for this matrix. A plain sum of revenue and cost lines is not meaningful. Category subtotals stay on.
- Conditional formatting on `Variance $` and `Variance Status`: Font color > Format style **Field value** > `Variance Color`.
- Below the matrix, a row of cards for the subtotals the matrix cannot show: `Gross Margin % Actual`, `Gross Margin % Budget`, `EBITDA Actual`, `EBITDA Budget`, `EBITDA Var $`.
- Slicer: `dim_Product[Product]` (affects revenue lines only; cost lines are "Unallocated").

### Page 3: Variance Drivers

| Visual | Fields |
|---|---|
| Table: price / volume by product | `dim_Product[Product]` (filter out Unallocated), `Units Budget`, `Units Actual`, `Price Budget`, `Price Actual`, `Volume Variance`, `Price Variance`, `Revenue Var $` |
| Clustered bar: cost variance by line | Y `dim_Account[Account]`, X `Variance $`; visual filter Category = COGS, Opex; data colors by rule using `Variance Color` |
| Line: units by product and month | X `MonthYear`, Y `Units Actual` and `Units Budget`, small multiples `dim_Product[Product]` (shows the Home Textiles stockout in May-June) |
| Table: commentary | `tbl_Commentary[Account]`, `Variance $`, `DriverType`, `Commentary`, `ForecastTreatment`. Clicking a bar in the cost chart filters this table |

### Page 4: Full-Year Outlook

| Visual | Fields |
|---|---|
| 4 cards | `Revenue Forecast FY` (ref: `Revenue FY Var %`), `Revenue Budget FY`, `EBITDA Forecast FY` (ref: `EBITDA FY Var %`), `EBITDA Budget FY` |
| Matrix | Rows CategoryName > Account; Values `Budget FY`, `Forecast FY`, `FY Variance $`; grand total off |
| Stacked column: forecast composition | X `MonthYear`, Y `EBITDA Forecast FY`, Legend: a calculated column on `dim_Date`: `Period Type = IF ( dim_Date[IsClosed] = 1, "Actual", "Forecast" )` |
| Waterfall | Category `CategoryName`, Y `FY Variance $` |

Do not put a Month slicer on this page.

## 6. Validation: Power BI must match Excel

With no slicers selected (YTD through Sep-26):

| Measure | Expected |
|---|---|
| `Revenue Actual` | 15,479,960 |
| `Revenue Budget` | 15,321,600 |
| `EBITDA Actual` | 1,532,618 |
| `EBITDA Budget` | 1,757,284 |
| `Variance $` (all accounts) | -224,666 |
| `Volume Variance` | 143,748 |
| `Price Variance` | 14,612 |
| `Revenue Forecast FY` | 23,440,686 |
| `EBITDA Forecast FY` | 3,060,196 |
| `EBITDA Budget FY` | 3,353,650 |

These are the values in the workbook as delivered. If you change any assumption in Excel, the Excel BvA tab becomes the reference.

## 7. Month-end refresh

1. Enter the new month on the `Actuals` tab and set `Assumptions!B6` to the new month count.
2. On `pbi_fact_PnL` and `pbi_fact_Units`, add the new month's Actual rows by copying the pattern of the last Actual month (the tables expand automatically).
3. Save the workbook, then **Refresh** in Power BI.

## Using the CSV fallback

If the Excel import gives trouble, use **Get data > Folder** or **Text/CSV** on the `csv/` files. Table and column names are identical, so every step above still applies. The CSVs are a snapshot and do not update when the workbook changes.
