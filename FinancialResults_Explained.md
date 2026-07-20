# `FinancialResults` — Full Code-Level Explanation

This section documents **only** the `FinancialResults` field, traced end-to-end through `views.py` — from where it's triggered in the main API view down through every helper function, data source, and fallback path.

---

## 1. Where it's triggered (main view function)

```python
result["data"]["FinancialResults"] = {}
financial_status_code = 201
if financial_status_code != 200:
    try:
        nse_fin_data = get_nse_new_financial(fin_code, scrip_code, isin_val)
        result["data"]["FinancialResults"] = nse_fin_data
    except Exception as e:
        result["data"]["FinancialResults"] = [ {... all fields "Null" ...} ]
```

- `financial_status_code` is **hard-coded to `201`**, not computed from any real status check. Since the condition is `!= 200`, this `if` block is effectively **always `True`** — `get_nse_new_financial()` always runs. This looks like leftover/dead logic (probably meant to gate on a real API status code at some point) rather than an intentional conditional — worth flagging to your team, since it reads like a bug even though it doesn't currently break anything.
- If `get_nse_new_financial()` throws **any** exception, `FinancialResults` falls back to a single-item list where every field is the string `"Null"` (the full field list is shown in Section 5).
- On success, `FinancialResults` is whatever `get_nse_new_financial()` returns — which is a **list of 1 or 2 objects** (see Section 4.6).

---

## 2. The core function: `get_nse_new_financial(fin_code, scrip_code, isin_val)`

This function builds the standalone and/or consolidated financial statement objects for a company. It pulls from **up to 5 different underlying tables**, chosen based on data availability, and merges them into a single normalized schema.

### 2.1 Step 1 — Try the "NSE format" P&L data first

```python
st_data_list = ace_balancesheet_results_ind_as_format_data_merged_ace_financial.objects.filter(Fincode=fin_code, nature="S")
cl_data_list = ace_balancesheet_results_ind_as_format_data_merged_ace_financial.objects.filter(Fincode=fin_code, nature="C")
```

- `nature="S"` → **Standalone** P&L figures; `nature="C"` → **Consolidated** P&L figures.
- Fields pulled: `Net_Sales`, `Other_Income`, `Interest`, `Depreciation`, `Pbt`, `Tax`, `Shares_Associate`, `Discontinued_Op`, `Pat`, `eps_basic`, `Inventory`, `Cash_bank`, plus `Date_End` and `Result_Type`.
- The **latest** record for each (`st_latest_dict`, `cl_latest_dict`) is picked using `max(..., key=lambda: int(Date_End))`.
- `nse_flag` is set to `"True"` if **either** standalone or consolidated data was found. This flag decides which "branch" of the whole function runs next.

**In plain terms:** this table is the primary/preferred source — company financials already reported/merged in NSE's standard format.

### 2.2 Step 2 — Branch A: `nse_flag == "True"` → pull the matching Balance Sheet data

If P&L data *was* found in Step 1, the function assumes real financial data exists for this company and fetches the corresponding **balance sheet** from a companion table:

```python
ace_balancesheet_result_balancesheet.objects.filter(FinCode=fin_code, nature="S")  # standalone
ace_balancesheet_result_balancesheet.objects.filter(FinCode=fin_code, nature="C")  # consolidated
```

Fields pulled here (28 total) cover both liabilities and assets, e.g. `Share_capital`, `Other_Equity_Reserve`, `Total_debts`, `ST_TradePayables`, `Net_block`, `Debtors`, `Cash_bank`, `Inventory`, `Currant_assets` [sic], `Current_laib_prov`, etc.

- The **latest** record (by `Date_end`, descending sort) is kept as `ace_balancesheet_data_st` / `ace_balancesheet_cons`.
- If no record exists for either nature, a **default dict of all `"Null"`** (28 keys) is used in its place — so the function never crashes on a missing balance sheet, it just fills with nulls.

### 2.3 Step 3 — Branch B: `nse_flag == "False"` → fall back to Capitaline data

If **no** P&L data was found at all in Step 1 (company isn't in the "NSE format" table), the function looks for an **alternate data vendor's code** — a `CapitalineCode` — to use a completely different source table:

```python
new_mapping.objects.filter(FINCODE=fin_code).values("CapitalineCode")
# if not found, retry by SCRIPCODE, then by ISIN
```

- If a `CapitalineCode` is found, the function queries `capitaline_new_download` (filtered by `NatureReport="S"` / `"C"`) for balance-sheet-style fields such as `ShareCapital`, `ReservesAndSurplus`, `NCL_Long_Term_Borrowings`, `TCA_Inventories`, `TCA_CashAndCashEquivalents`, `TotalCurrentAssets`, etc.
- Again, the **latest record by `YearEnd`** is kept (`captline_data_st` / `captline_data_cons`), with an all-`"Null"` default if nothing is found.
- If no `CapitalineCode` can be resolved at all, `capitaline_code` stays `"Null"` and this whole block is skipped — the function has no data for the company beyond whatever step 4 can still recover.

**In plain terms:** Capitaline is the **secondary/legacy source**, used only when the company has no data in the primary NSE-format tables.

### 2.4 Step 4 — Assembling `standalone_result` and `consolidated_result`

Two helper functions build the final per-nature objects, and which one is used depends on **whether a `CapitalineCode` was resolved**:

| Condition | Function used | Data source for balance-sheet fields |
|---|---|---|
| `capitaline_code != "Null"` | `nse_fin_assign()` | `capitaline_new_download` fields (`ShareCapital`, `ReservesAndSurplus`, `TCA_Inventories`, …) |
| `capitaline_code == "Null"` | `nse_fin_assign_bs()` | `ace_balancesheet_result_balancesheet` fields (`Share_capital`, `Other_Equity_Reserve`, `Current_Investments`, …) |

Both functions return the **same output schema** (so the consumer of the API never sees a difference), just sourced from different underlying tables. Key differences worth knowing:

- **`nse_fin_assign_bs()` computes a few fields by summing two source columns**, rather than a 1:1 mapping:
  - `noncurrentInvestmentsHalfYearEndingActualsCurrentHy` = `Investments_NF + Invest_Sub_JV_Associates`
  - `cashHalfYearEndingActualsCurrentHy` = `Cash_bank + Bank_Bal_Other_CandCEq`
  - `intangibleAssetsHalfYearEndingActualsCurrentHy` = `Other_Intangible_Assets + Goodwill`
- **`nse_fin_assign()`** (Capitaline path) does **not** sum anything — it's a straight field-to-field mapping.
- In **both** functions, all of the *P&L* fields (`netSalesQuarterlyActualsCurrentQt`, `epsQuarterlyActualsCurrentQt`, `patYtdActualsCurrentQt`, etc.) and all *cash flow* fields are initialized to `"Null"` — they are **not** populated here. That happens later, in Step 6 (cash flow) and Step 7 (YTD/quarterly).
- `financialYear` is computed by a separate helper, `nse_financial_year_1()` (see Section 3), which returns a human-readable range like `"01 Apr 2023 - 31 Mar 2024"`.

### 2.5 Step 4b — Patch-up for `inventories` and `cash` if still `"Null"`

After building `standalone_result` / `consolidated_result`, the code double-checks two specific fields and re-patches them directly from the raw source record if they came back `"Null"`:

```python
if consolidate_result["inventoriesHalfYearEndingActualsCurrentHy"] == "Null":
    consolidate_result["inventoriesHalfYearEndingActualsCurrentHy"] = captline_data_cons["TCA_Inventories"] ...
if consolidate_result["cashHalfYearEndingActualsCurrentHy"] == "Null":
    consolidate_result["cashHalfYearEndingActualsCurrentHy"] = captline_data_cons["TCA_CashAndCashEquivalents"] ...
```

(The equivalent patch happens for `standalone_result` too, and for both the Capitaline and ace_balancesheet variants.) This is a safety net for cases where the primary mapping missed the value but the raw record actually has it.

### 2.6 Step 5 — Enriching with Cash Flow Statement data

Cash flow figures are pulled **separately**, from two possible table pairs (an older table and a newer fallback table), each split by Standalone/Consolidated:

| Purpose | Primary table | Fallback table (used only if primary is empty) |
|---|---|---|
| Standalone cash flow | `ace_balancesheet_result_cashflow` (`nature="S"`) | `ace_financial_cf` (`type="S"`) |
| Consolidated cash flow | `ace_balancesheet_result_cashflow` (`nature="C"`) | `ace_financial_cf` (`type="C"`) |

For whichever source is available, these fields on `standalone_result` / `consolidated_result` get overwritten:

- `netCashFromOperationsHalfYearlyActualsCurrentHy`
- `netCashFromInvestingHalfYearlyActualsCurrentHy`
- `netCashFromFinanceHalfYearlyActualsCurrentHy`
- `totalCashFlowBeforeEffectOfExchangeRateChanges` — computed as `Net_Cash_inflow_outflow − Effect_foreign_exchange`
- `totalCashFlowAfterEffectOfExchangeRateChanges` — **only computed/set for the consolidated result.** For the standalone result, this field is never assigned after the initial `"Null"` default from Step 4 — meaning **`totalCashFlowAfterEffectOfExchangeRateChanges` will always show `"Null"` for the Standalone entry.** This is a real behavioral quirk in the current code (likely an oversight — the standalone branch sets `"totalCashFlowBeforeEffectOfExchangeRateChanges"` twice instead of also setting the "After" field), worth flagging if the field is expected to be populated for both natures.

### 2.7 Step 6 — Assembling the final list

```python
if cl_data_list != [] and st_data_list != []:
    result = [standalone_result, consolidate_result]
elif st_data_list == []:
    result = [consolidate_result]
elif cl_data_list == []:
    result = [standalone_result]
```

- If **both** Standalone and Consolidated P&L data exist → `FinancialResults` is a **2-item list**: `[standalone, consolidated]`.
- If only one exists → it's a **1-item list** containing just that one.
- (Edge case: if the Capitaline-only path was used and *neither* `st_data_list` nor `cl_data_list` had NSE-format P&L rows, `standalone_result`/`consolidate_result` may be undefined — this would raise a `NameError`, which propagates up and is caught by the outer `try/except` in the main view, causing the all-`"Null"` fallback described in Section 1.)

### 2.8 Step 7 — YTD & Quarterly P&L figures

```python
result = ytd_calculations(fin_code, result)
```

This is where the **P&L fields left as `"Null"`** in Step 4 (`netSalesQuarterlyActualsCurrentQt`, `netSalesYtdActualsCurrentQt`, `otherIncomeQuarterlyActualsCurrentQt`, `epsQuarterlyActualsCurrentQt`, `patQuarterlyActualsCurrentQt`, `taxQuarterlyActualsCurrentQt`, `interestQuarterlyActualsCurrentQt`, `depreciationAmortisationQuarterlyActualsCurrentQt`, and their YTD equivalents) actually get filled in.

- `ytd_calculations()` is **imported from `app.functions`**, not defined in `views.py` itself, so its internal logic isn't visible in this file — if you need the exact YTD computation method (e.g. how quarterly is derived vs YTD, which quarter is "current"), that function needs to be reviewed separately.
- If this call throws an exception, it's caught silently (`except Exception as e: print(...); pass`) and `result` is left as-is (i.e., those P&L fields stay `"Null"`).

### 2.9 Step 8 — Final unit conversion pass

```python
for data_dict in result:
    for key, value in data_dict.items():
        if key in ("epsQuarterlyActualsCurrentQt", "epsYtdActualsCurrentQt"):
            data_dict[key] = str(value)      # EPS is left as-is, just stringified
            continue
        if value in ("Null", None):
            continue
        if capitaline_code != "Null":
            if is_float(value):
                data_dict[key] = str(float(value) * 10000000)   # ×1 crore
        else:
            if is_float(value):
                data_dict[key] = str(float(value) * 1000000)    # ×10 lakh
```

- **Every numeric field except EPS** gets multiplied at the very end, and the multiplier **depends on which data source was used**:
  - Company resolved via **Capitaline** (`capitaline_code != "Null"`) → multiply by **10,000,000** (1 crore).
  - Company resolved via **NSE/ace tables** (`capitaline_code == "Null"`) → multiply by **1,000,000** (10 lakh).
- This implies the two raw source systems store figures in **different base units** (Capitaline likely in ₹ Crore, ACE tables likely in ₹ Lakh), and this final step normalizes both onto the same absolute-Rupee scale before returning them in the response.
- **EPS fields are exempt** from this multiplication — they're per-share values, not a monetary total, so scaling them the same way would be incorrect.
- All values are returned as **strings**, even though they're numeric, for schema consistency.

---

## 3. `nse_financial_year_1()` — how `financialYear` is derived

This helper (not part of the main data-fetch flow, but called from within `nse_fin_assign`/`nse_fin_assign_bs`) computes the human-readable financial year range shown in `financialYear`, e.g.:

```
"01 Apr 2023 - 31 Mar 2024"
```

At a high level it:
1. Resolves the company's `CapitalineCode` again (independently) for context.
2. Finds the most recent "year end" date from `ace_balancesheet` (or `ace_financial_cf` / `capitaline_new_download` as fallbacks) — this anchors the **end** of the FY.
3. Finds the most recent **reporting** date from `ace_balancesheet_results_ind_as_format_data_merged_ace_financial` filtered to annual records (`NoOfMonths` of `12` or `15`) — this anchors what's actually being reported.
4. Compares the two years/months and builds the `"01 {startMonth} {startYear} - {lastDayOfMonth} {endMonth} {endYear}"` string, handling edge cases for December year-ends, June year-ends, and mismatched years.
5. Returns `None` (→ becomes `"Null"` in the JSON, since `None` is JSON `null` unless already cast to string) if no usable dates are found, or if the internal comparison logic raises a `ValueError`.

This function has fairly involved branching (calendar month names, day-count lookups, year-diff checks) — if the FY range ever looks wrong for a particular company, this is the function to inspect first.

---

## 4. Data source summary (all tables touched by `FinancialResults`)

| Table | Used for | Notes |
|---|---|---|
| `ace_balancesheet_results_ind_as_format_data_merged_ace_financial` | P&L figures (Net Sales, PAT, EPS, Tax, Interest, Depreciation, Inventory, Cash) | Primary/preferred source; presence of data here sets `nse_flag = "True"` |
| `ace_balancesheet_result_balancesheet` | Balance sheet figures | Used only when `nse_flag == "True"` (i.e., primary P&L source has data) |
| `new_mapping` | Resolves `CapitalineCode` from `FINCODE` / `SCRIPCODE` / `ISIN` | Only queried when `nse_flag == "False"` |
| `capitaline_new_download` | Balance sheet figures (alternate/legacy vendor) | Used only when `CapitalineCode` was successfully resolved |
| `ace_balancesheet_result_cashflow` | Cash flow figures | Primary cash flow source |
| `ace_financial_cf` | Cash flow figures, financial year lookups | Fallback cash flow source; also used in FY-range calculation |
| `ace_balancesheet` | Year-end date for FY range calculation | Used only inside `nse_financial_year_1()` |
| *(external)* `ytd_calculations()` in `app/functions.py` | Fills in quarterly/YTD P&L fields | Not visible in `views.py` — review separately if needed |

---

## 5. Full field reference (final output schema)

Every object inside the `FinancialResults` list (whether Standalone or Consolidated) has this shape:

| Field | Meaning | Populated by |
|---|---|---|
| `reportingQuarter` | Reporting quarter label | Always `"Null"` in current code — never assigned a real value anywhere in this flow |
| `natureOfReport` | `"Standalone"` or `"Consolidated"` | Set directly in `nse_fin_assign`/`nse_fin_assign_bs` |
| `financialYear` | FY date range string | `nse_financial_year_1()` (Section 3) |
| `netSalesQuarterlyActualsCurrentQt` / `netSalesYtdActualsCurrentQt` | Net sales, quarterly / year-to-date | `ytd_calculations()` (Section 2.8) |
| `otherIncomeQuarterlyActualsCurrentQt` / `otherIncomeYtdActualsCurrentQt` | Other income, quarterly / YTD | `ytd_calculations()` |
| `epsQuarterlyActualsCurrentQt` / `epsYtdActualsCurrentQt` | Earnings per share, quarterly / YTD (not unit-scaled) | `ytd_calculations()` |
| `patQuarterlyActualsCurrentQt` / `patYtdActualsCurrentQt` | Profit after tax, quarterly / YTD | `ytd_calculations()` |
| `taxQuarterlyActualsCurrentQt` / `taxYtdActualsCurrentQt` | Tax expense, quarterly / YTD | `ytd_calculations()` |
| `interestQuarterlyActualsCurrentQt` / `interestYtdActualsCurrentQt` | Interest expense, quarterly / YTD | `ytd_calculations()` |
| `depreciationAmortisationQuarterlyActualsCurrentQt` / `...YtdActualsCurrentQt` | Depreciation & amortisation, quarterly / YTD | `ytd_calculations()` |
| `netCashFromOperationsHalfYearlyActualsCurrentHy` | Net cash from operating activities | Cash flow tables (Section 2.6) |
| `netCashFromInvestingHalfYearlyActualsCurrentHy` | Net cash from investing activities | Cash flow tables |
| `netCashFromFinanceHalfYearlyActualsCurrentHy` | Net cash from financing activities | Cash flow tables |
| `totalCashFlowBeforeEffectOfExchangeRateChanges` | Net cash flow before FX effect | Computed: `Net_Cash_inflow_outflow − Effect_foreign_exchange` |
| `effectOfExchangeRateChanges` | FX translation effect | Always `"Null"` — never explicitly assigned in this flow |
| `totalCashFlowAfterEffectOfExchangeRateChanges` | Net cash flow after FX effect | Only set for **Consolidated**; **Standalone always returns `"Null"`** (see 2.6 quirk) |
| `shareCapitalHalfYearEndingActualsCurrentHy` | Share capital | Balance sheet table (Capitaline or ACE, per Section 2.4) |
| `reservesAndSurplusHalfYearEndingActualsCurrentHy` | Reserves & surplus | Balance sheet table |
| `pendingAllotmentHalfYearEndingActualsCurrentHy` | Share application money pending allotment | Balance sheet table |
| `longTermBorrowingsHalfYearEndingActualsCurrentHy` | Long-term borrowings | Balance sheet table |
| `deferredTaxLiabilitiesHalfYearEndingActualsCurrentHy` | Deferred tax liabilities (net) | Balance sheet table |
| `longTermProvisionHalfYearEndingActualsCurrentHy` | Long-term provisions | Balance sheet table |
| `totalNonCurrentLiabilitiesHalfYearEndingActualsCurrentHy` | Total non-current liabilities | Balance sheet table |
| `shortTermBorrowingsHalfYearEndingActualsCurrentHy` | Short-term borrowings | Balance sheet table |
| `tradePayablesHalfYearEndingActualsCurrentHy` | Trade payables | Balance sheet table |
| `shortTermProvisionHalfYearEndingActualsCurrentHy` | Short-term provisions | Balance sheet table |
| `subtotalCurrentLiabilitiesHalfYearEndingActualsCurrentHy` | Subtotal — current liabilities | Balance sheet table |
| `fixedAssetsHalfYearEndingActualsCurrentHy` | Net fixed assets (incl. CWIP) | Balance sheet table |
| `noncurrentInvestmentsHalfYearEndingActualsCurrentHy` | Non-current investments | Balance sheet table (summed field in `nse_fin_assign_bs`, see 2.4) |
| `longtermadvancesHalfYearEndingActualsCurrentHy` | Long-term loans & advances | Balance sheet table |
| `tradeReceivablesMoreThan6MonthsHalfYearEndingActualsCurrentHy` | Trade receivables (>6 months / non-current) | Balance sheet table |
| `subtotalNonCurrentAssetsHalfYearEndingActualsCurrentHy` | Subtotal — non-current assets | Balance sheet table |
| `currentInvestmentsHalfYearEndingActualsCurrentHy` | Current investments | Balance sheet table |
| `inventoriesHalfYearEndingActualsCurrentHy` | Inventories | Balance sheet table, with patch-up from raw record if null (2.5) |
| `tradeReceivablesHalfYearEndingActualsCurrentHy` | Trade receivables (current) | Balance sheet table |
| `cashHalfYearEndingActualsCurrentHy` | Cash & bank balances | Balance sheet table (summed field in `nse_fin_assign_bs`, see 2.4), with patch-up (2.5) |
| `shortTermAdvancesHalfYearEndingActualsCurrentHy` | Short-term loans & advances | Balance sheet table |
| `subtotalCurrentAssetsHalfYearEndingActualsCurrentHy` | Subtotal — current assets | Balance sheet table |
| `intangibleAssetsHalfYearEndingActualsCurrentHy` | Intangible assets | Balance sheet table (summed field in `nse_fin_assign_bs`, see 2.4) |
| `deferredTaxAssetsHalfYearEndingActualsCurrentHy` | Deferred tax assets (net) | Balance sheet table |
| `inventoryHalfYearEndingActualsCurrentHy` | Inventory (duplicate of `inventories...` above, from the P&L-side latest record instead of balance sheet) | `latest_dict["Inventory"]` — note this is a **separate near-duplicate field** from `inventoriesHalfYearEndingActualsCurrentHy` |
| `tcaHalfYearEndingActualsCurrentHy` | Total Current Assets | Balance sheet table |
| `tclHalfYearEndingActualsCurrentHy` | Total Current Liabilities | Balance sheet table |

**Naming quirk worth documenting:** `inventoriesHalfYearEndingActualsCurrentHy` and `inventoryHalfYearEndingActualsCurrentHy` are **two distinct fields** that both mean "inventory" but are sourced slightly differently (one from the balance sheet/Capitaline record, one from the P&L-side latest record's `Inventory` column). They usually carry the same or very similar value, but they are not guaranteed to always match.

---

## 6. Quick summary (TL;DR for the README)

`FinancialResults` returns a list of 1–2 objects (Standalone and/or Consolidated) containing the company's latest quarterly/YTD P&L figures, half-yearly balance sheet position, and half-yearly cash flow — pulled from NSE-format tables when available, or Capitaline data as a fallback, with a final unit-normalization step so all monetary fields come back on a consistent absolute-Rupee scale (except EPS, which is left as a per-share value). Known quirks: `reportingQuarter` and `effectOfExchangeRateChanges` are never populated, `totalCashFlowAfterEffectOfExchangeRateChanges` is only populated for the Consolidated entry, and there are two separate inventory fields that may not always match.
