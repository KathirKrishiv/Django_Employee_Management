## `data` Object — Field Descriptions

### `bseStatus`
- **Type:** `string`
- **Description:** Indicates whether the company's security is currently sub-listed / active on the **BSE (Bombay Stock Exchange)**.
- **Source:** `ace_company_master.bse_sublisting`, looked up using `scripcode`. As a fallback (when `scrip_code` is `"Null"`), it is instead read from `bse_security_list_2_9_24.Status` using the BSE `security_code`.
- **Possible values:** e.g. `"Active"`, `"Suspended"`, `"Delisted"`, or `"Null"` if no record is found / an error occurs while fetching it.
- **Used for:** Also feeds into the `listedWith` field — if `bseStatus` has a real (non-null) value, the company is considered listed on BSE.

### `nseStatus`
- **Type:** `string`
- **Description:** Indicates whether the company's security is currently sub-listed / active on the **NSE (National Stock Exchange)**.
- **Source:** `ace_company_master.nse_sublisting`, looked up using `fin_code`.
- **Possible values:** e.g. `"Active"`, `"Suspended"`, `"Delisted"`, or `"Null"` if no record is found / an error occurs while fetching it.
- **Used for:** Also feeds into the `listedWith` field, similar to `bseStatus`.

> **Note:** `listedWith` in the output (`"BSE & NSE"`, `"BSE"`, `"NSE"`, or `"Null"`) is derived by checking whether `bseStatus` and/or `nseStatus` hold real values.

---

### `EquityData`
- **Type:** `object`
- **Description:** Latest available equity/share-price snapshot for the company, based on the most recent `price_date` record.
- **Source:** `ace_equity` table, filtered by `fin_code`, sorted ascending by `price_date` (latest record picked via `[-1]`).
- **Sub-fields:**
  | Field | Description |
  |---|---|
  | `clsPric` | Latest closing price of the stock |
  | `stkExchange` | Exchange the latest price was recorded on (`BSE` / `NSE`) |
  | `tradeDate` | Date of the latest recorded price (`YYYY-MM-DD`) |
  | `pe` | Trailing twelve month P/E ratio (`ttmpe`) |
  | `faceValue` | Face value of the share (`fv`) |
  | `faceValueDate` | Date corresponding to the face value / price record |
- **Fallback:** If no equity data is found, all sub-fields default to `"Null"`.

---

### `52WeekHL`
- **Type:** `object`
- **Description:** 52-week high/low price data for the stock on both exchanges.
- **Source:** `ace_52whl` table, filtered by `scripcode` (or `symbol` as a fallback if no record is found by scrip code).
- **Structure:**
```json
{
  "bse": {
    "bseHigh": "...",
    "bseHighDate": "...",
    "bseLow": "...",
    "bseLowDate": "..."
  },
  "nse": {
    "nseHigh": "...",
    "nseHighDate": "...",
    "nseLow": "...",
    "nseLowDate": "..."
  }
}
```
- **Fallback:** If lookup fails/errors, all values under `bse` and `nse` default to `"Null"`.

---

### `MarketCapitalData`
- **Type:** `object`
- **Description:** Latest market capitalization of the company, computed from the `mcap` value in the equity table and converted to **₹ Crores** (divided by `10,000,000` and rounded to 2 decimals).
- **Source:** Derived from the same `ace_equity` record used for `EquityData` (`mcap`, `stk_exchange`, `price_date`).
- **Logic:** Which branch runs depends on `EquityData.stkExchange` (`BSE` or `NSE`); if the exchange is neither, or an error occurs, defaults/errors are returned.
- **Sub-fields:**
  | Field | Description |
  |---|---|
  | `marketCapValue` | Market capitalization in ₹ Crores (as a string) |
  | `stkExchange` | Exchange the market cap value is based on |
  | `tradeDate` | Date corresponding to the market cap value |
- **Fallback:** `"Null"` values for each field, or `{"Error": "No Record Found"}` if the whole block fails.

---

### `FinancialResults`
- **Type:** `object` / `list of objects`
- **Description:** Quarterly and half-yearly financial results of the company (P&L, balance sheet, and cash flow statement figures), primarily sourced from NSE financial filings.
- **Source:** `get_nse_new_financial(fin_code, scrip_code, isin_val)` function; internally reads and unit-converts values from `capitaline_new_download` records (values are multiplied by `10,000,000` or `1,000,000` depending on the field, to normalize units).
- **Key sub-field groups:**
  - **Report meta:** `reportingQuarter`, `natureOfReport`, `financialYear`
  - **Quarterly P&L (Actuals – Current Quarter):** `netSalesQuarterlyActualsCurrentQt`, `otherIncomeQuarterlyActualsCurrentQt`, `epsQuarterlyActualsCurrentQt`, `patQuarterlyActualsCurrentQt`, `taxQuarterlyActualsCurrentQt`, `interestQuarterlyActualsCurrentQt`, `depreciationAmortisationQuarterlyActualsCurrentQt`, etc.
  - **Year-to-Date (YTD) equivalents** of the above quarterly fields (e.g. `netSalesYtdActualsCurrentQt`).
  - **Half-Yearly Cash Flow:** `netCashFromOperationsHalfYearlyActualsCurrentHy`, `netCashFromInvestingHalfYearlyActualsCurrentHy`, `netCashFromFinanceHalfYearlyActualsCurrentHy`, `totalCashFlowBeforeEffectOfExchangeRateChanges`, `effectOfExchangeRateChanges`, `totalCashFlowAfterEffectOfExchangeRateChanges`
  - **Half-Yearly Balance Sheet (Liabilities & Assets, "…HalfYearEndingActualsCurrentHy"):** e.g. `shareCapital…`, `reservesAndSurplus…`, `longTermBorrowings…`, `tradePayables…`, `fixedAssets…`, `inventories…`, `tradeReceivables…`, `cash…`, subtotal/total fields such as `subtotalCurrentLiabilities…`, `subtotalNonCurrentAssets…`, `tcaHalfYearEndingActualsCurrentHy` (Total Current Assets), `tclHalfYearEndingActualsCurrentHy` (Total Current Liabilities).
- **Fallback:** If `get_nse_new_financial` fails, a default list containing one object with all of the above fields set to `"Null"` is returned instead.

---

### General Notes
- Any field that could not be fetched (missing record, DB error, exception) is uniformly represented as the string `"Null"` (not JSON `null`) throughout this API, to keep the response schema consistent for consumers.
- `scrip_code` is the BSE identifier; `fin_code` is the internal ACE financial code; `isin` is the security's ISIN — these are the primary keys used to cross-reference `bseStatus`, `nseStatus`, `EquityData`, `52WeekHL`, `MarketCapitalData`, and `FinancialResults` across different underlying tables.
