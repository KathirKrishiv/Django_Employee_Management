# Marketdata API Result

The following fields are returned under the `result.data` object in Marketdata API result.

## Directly Retrieved Data's

**Source Table:** `new_mapping`  
**Lookup Fields:** `PAN`, `priority`

### Response Structure

| Field | Type     | Description | Source |
|-------|----------|-------------|--------|
| `nameofTheCompany` | String | Official name of the company. | `new_mapping.COMPNAME` |
| `scripCode` | String | BSE Scrip Code. | `new_mapping.SCRIPCODE` |
| `pan` | String | PAN supplied in the API request. | Request Parameter |
| `symbol` | String | Trading symbol. Retrieved from new_mapping.SYMBOL; if unavailable, the value is retrieved from ace_company_master.symbol. | `new_mapping.SYMBOL`, `ace_company_master.symbol` |
| `isin` | String | ISIN of the company. | `new_mapping.ISIN` |
| `listedWith` | String | Exchange(s) where the company is listed (`BSE`, `NSE`, `BSE & NSE`, `Null`). | Derived from `new_mapping.Bse_sublisting` and `new_mapping.Nse_sublisting` |

---

## `bseStatus`

| Property | Value |
|----------|-------|
| Type | String |
| Description |  Current BSE listing status of the security. |
| Source Table | `ace_company_master` |
| Lookup Column | `scripcode` |
| Returned Column | `bse_sublisting` |
| Possible Values | `Active`, `Suspended`, `Delisted`, `Null` |

---

## `nseStatus`

| Property | Value |
|----------|-------|
| Type | String |
| Description | Current NSE listing status of the security. |
| Source Table | `ace_company_master` |
| Lookup Column | `fincode` |
| Returned Column | `nse_sublisting` |
| Possible Values | `Active`, `Suspended`, `Delisted`, `Null` |
---

## `EquityData`
- **Type:** `object`
- **Description:**
  - The EquityData object contains the latest available equity market information for the company. The data is retrieved from the ace_equity table using the company's FINCODE.
  - If multiple equity records exist, the API sorts them by `price_date` in ascending order and returns the most recent record.


### Source
|Source Table	|Lookup Column	|Sort Column	  |           Selected Record|
|-------------|-------------- |------------    |         -----------------|
|ace_equity	  |  fincode	    | price_date (Ascending)	| Latest (price_date)|


### Response Structure

| Field | Type | Description | Source Column |
|-------|------|-------------|---------------|
| `clsPric` | Number / String | Latest closing market price of the equity. Returns `"Null"` if unavailable. | `price` |
| `stkExchange` | String | Stock exchange on which the latest equity price was recorded (BSE/NSE). | `stk_exchange` |
| `tradeDate` | Date | Trade date corresponding to the latest closing price. | `price_date` |
| `pe` | Number / String | Trailing Twelve Months (TTM) Price-to-Earnings ratio. Returns `"Null"` if unavailable. | `ttmpe` |
| `faceValue` | Number / String | Face value of the equity share. | `fv` |
| `faceValueDate` | Date | Date corresponding to the face value information. | `price_date` |

### Retrieval Logic

The API retrieves the latest available equity information for the company using the company's **FINCODE**.

#### Step 1 - Retrieve Equity Records

```
Lookup table - ace_equity using:

1. fincode
```

#### Step 2 - Select Latest Record

```
If one or more records are found:

1. Sort the records by `price_date` in ascending order.
2. Select the most recent record (latest `price_date`).
```

#### Step 3 - Populate EquityData

The following fields are populated from the selected record:

- `clsPric` ← `price`
- `stkExchange` ← `stk_exchange`
- `tradeDate` ← `price_date` (returned in `YYYY-MM-DD` format)
- `pe` ← `ttmpe`
- `faceValue` ← `fv`
- `faceValueDate` ← `price_date` (returned in `YYYY-MM-DD` format)

#### Step 4 - Update Listing Exchange

Based on the retrieved stock exchange:

- If `stkExchange` is `BSE`, `listedWith` is updated to `BSE`.
- If `stkExchange` is `NSE`, `listedWith` is updated to `NSE`.


### Null Handling

If no matching equity record is found or an exception occurs:

- `clsPric` = `"Null"`
- `stkExchange` = `"Null"`
- `tradeDate` = `"Null"`
- `pe` = `"Null"`
- `faceValue` = `"Null"`
- `faceValueDate` = `"Null"`

If the latest record exists but the `ttmpe` value is `NULL` in the database:

- `pe` is returned as `"Null"`.


### Notes

- Equity data is retrieved from the `ace_equity` table using the company's **FINCODE**.
- If multiple equity records exist, the API sorts them by `price_date` in ascending order and returns the latest available record.
- `tradeDate` and `faceValueDate` are derived from `price_date` and returned in **YYYY-MM-DD** format.
- The `listedWith` field is updated based on the value of `stkExchange`.
- Database `NULL` values are returned as the string `"Null"`.
- Any unexpected exception during retrieval results in `"Null"` values for the corresponding fields.
  
---

## 52WeekHL

**Description:**
- The `52WeekHL` object contains the company's **52-week High and Low prices** for both the **Bombay Stock Exchange (BSE)** and the **National Stock Exchange (NSE)**.
- The data is retrieved from the `ace_52whl` table.

### Source

| Source Table | Primary Lookup | Fallback Lookup |
|--------------|----------------|-----------------|
| `ace_52whl` | `scripcode` | `symbol` |

If the `scripCode` is not available, the API retrieves the data using:

| Source Table | Primary Lookup | Fallback Lookup |
|--------------|----------------|-----------------|
| `ace_52whl` | `fincode` | `symbol` |


### Response Structure

| Field | Type | Description | Source Column |
|------|------|-------------|---------------|
| `bseHigh` | Number / String | Highest price traded on BSE during the last 52 weeks. | `bse_high` |
| `bseHighDate` | Date | Date on which the 52-week high was recorded on BSE. | `bse_highdate` |
| `bseLow` | Number / String | Lowest price traded on BSE during the last 52 weeks. | `bse_low` |
| `bseLowDate` | Date | Date on which the 52-week low was recorded on BSE. | `bse_lowdate` |
| `nseHigh` | Number / String | Highest price traded on NSE during the last 52 weeks. | `nse_high` |
| `nseHighDate` | Date | Date on which the 52-week high was recorded on NSE. | `nse_highdate` |
| `nseLow` | Number / String | Lowest price traded on NSE during the last 52 weeks. | `nse_low` |
| `nseLowDate` | Date | Date on which the 52-week low was recorded on NSE. | `nse_lowdate` |


### Retrieval Logic

The API follows the lookup sequence below.

#### Case 1 - Scrip Code Available

```
Lookup table - ace_52whl using:

1. scripcode
2. If no record is found, lookup using symbol
```


#### Case 2 - Scrip Code Not Available

```
Lookup table - ace_52whl using:

1. fincode
2. If no record is found, lookup using symbol
```

If a record is found:

- `scripCode` is updated from `ace_52whl.scripcode`.
- `symbol` is updated from `ace_52whl.symbol`.
- `bseStatus` is refreshed using the retrieved `scripCode`.

### Null Handling

If no matching record is found or an exception occurs, the API returns Null for all fields:


### Notes

- All dates are returned in **YYYY-MM-DD** format.
- Database `NULL` values are returned as the string `"Null"`.
- When the `scripCode` is unavailable, the API automatically attempts to retrieve the data using `fincode`, and then `symbol`.
- If a valid `scripCode` is obtained during the fallback lookup, the API updates the response with the new `scripCode`, `symbol`, and corresponding `bseStatus`.

---

## MarketCapitalData

**Description:**
- The `MarketCapitalData` object provides the company's latest **Market Capitalization** details based on the most recent equity record available in the `ace_equity` table.

- The data is retrieved using the company's **FINCODE**.


### Source

| Source Table | Lookup Column |
|--------------|---------------|
| `ace_equity` | `fincode` |


### Response Structure

| Field | Type | Description | Source Column |
|------|------|-------------|---------------|
| `marketCapValue` | Number / String | Latest market capitalization value (in Crores). | `mcap` |
| `stkExchange` | String | Stock exchange associated with the market capitalization record. | `stk_exchange` |
| `tradeDate` | Date | Date on which the market capitalization value was recorded. | `price_date` |


### Retrieval Logic

1. Retrieve the equity records from `ace_equity` using the company's **FINCODE**.
2. Use the same equity record selected for the `EquityData` section.
3. Read the following columns:
   - `mcap`
   - `stk_exchange`
   - `price_date`
4. Convert the market capitalization into **Crores** using:

```text
Market Capitalization (Crores) = mcap / 10,000,000
```

5. Round the calculated value to **2 decimal places** before returning it in the response.

### Exchange Logic

The `MarketCapitalData` object is populated only when the stock exchange obtained from `EquityData.stkExchange` is either:

- `BSE`
- `NSE`

If the exchange is unavailable or contains any other value, the API returns `"Null"` for all fields.


### Null Handling

If the market capitalization data is unavailable or an exception occurs while retrieving the data, the API returns Null for all fields:


### Notes

- Market capitalization is retrieved from the `ace_equity.mcap` column.
- The value is converted from its stored unit into **Crores** by dividing it by **10,000,000**.
- The result is rounded to **2 decimal places**.
- The `tradeDate` is derived from `price_date` and returned in **YYYY-MM-DD** format.
- Database `NULL` values are returned as the string `"Null"`.

---

## `FinancialResults`

-   **Type:** `array<object>`
-   **Description:**
    -   The FinancialResults object contains the latest available
        financial information for the company.
    -   The API returns one or two records depending on data
        availability:
        -   **Standalone**
        -   **Consolidated**
    -   Financial data is primarily retrieved from NSE financial tables.
        If unavailable, the API falls back to Capitaline and ACE Balance
        Sheet data.

### Source

The financial data is retrieved from multiple source tables using the following lookup and fallback sequence.

| Source Table | Lookup Columns | Sort Order | Selected Record |
|--------------|----------------|------------|-----------------|
| `ace_balancesheet_results_ind_as_format_data_merged_ace_financial` | `FINCODE`, `nature (S/C)` | `Date_End` (Descending) | Latest record |
| `ace_balancesheet_result_balancesheet` | `FINCODE`, `nature (S/C)` | `Date_End` (Descending) | Latest record |
| `ace_balancesheet_result_cashflow` | `FINCODE`, `nature (S/C)` | Latest available | Latest record |
| `ace_financial_cf` | `FINCODE`, `type (S/C)` | `year_end` (Descending) | Fallback record |
| `new_mapping` | `FINCODE`, `SCRIPCODE`, `ISIN` | — | Retrieves `CapitalineCode` |
| `capitaline_new_download` | `CapitalineCode`, `NatureReport` | `YearEnd` (Descending) | Latest record |

#### Source Notes

- **S** = Standalone
- **C** = Consolidated
- The API first attempts to retrieve data from the ACE financial tables.
- If cash flow data is unavailable in the primary source, it falls back to `ace_financial_cf`.
- `new_mapping` is used to obtain the corresponding `CapitalineCode`.
- `capitaline_new_download` is queried using the retrieved `CapitalineCode` to fetch the latest available financial data.


### Response Structure

The API returns an array of objects. Each object contains the latest available financial information for a reporting period.

## Fields

| Field | Type | Description |
|-------|------|-------------|
| `reportingQuarter` | String | Reporting quarter. Returns `"Null"` if unavailable. |
| `natureOfReport` | String | Indicates whether the financial results are **Standalone** or **Consolidated**. |
| `financialYear` | String | Financial year period derived using the available year-end information. |
| `netSalesQuarterlyActualsCurrentQt` | Number / String | Quarterly Net Sales. |
| `netSalesYtdActualsCurrentQt` | Number / String | Year-to-date Net Sales. |
| `otherIncomeQuarterlyActualsCurrentQt` | Number / String | Quarterly Other Income. |
| `otherIncomeYtdActualsCurrentQt` | Number / String | Year-to-date Other Income. |
| `epsQuarterlyActualsCurrentQt` | Number / String | Quarterly Earnings Per Share (EPS). |
| `epsYtdActualsCurrentQt` | Number / String | Year-to-date Earnings Per Share (EPS). |
| `patQuarterlyActualsCurrentQt` | Number / String | Quarterly Profit After Tax (PAT). |
| `patYtdActualsCurrentQt` | Number / String | Year-to-date Profit After Tax (PAT). |
| `taxQuarterlyActualsCurrentQt` | Number / String | Quarterly Tax Expense. |
| `taxYtdActualsCurrentQt` | Number / String | Year-to-date Tax Expense. |
| `interestQuarterlyActualsCurrentQt` | Number / String | Quarterly Interest Expense. |
| `interestYtdActualsCurrentQt` | Number / String | Year-to-date Interest Expense. |
| `depreciationAmortisationQuarterlyActualsCurrentQt` | Number / String | Quarterly Depreciation & Amortisation Expense. |
| `depreciationAmortisationYtdActualsCurrentQt` | Number / String | Year-to-date Depreciation & Amortisation Expense. |
| `netCashFromOperationsHalfYearlyActualsCurrentHy` | Number / String | Net cash generated from operating activities. |
| `netCashFromInvestingHalfYearlyActualsCurrentHy` | Number / String | Net cash generated from investing activities. |
| `netCashFromFinanceHalfYearlyActualsCurrentHy` | Number / String | Net cash generated from financing activities. |
| `totalCashFlowBeforeEffectOfExchangeRateChanges` | Number / String | Total cash flow before the effect of exchange rate changes. |
| `effectOfExchangeRateChanges` | Number / String | Effect of exchange rate changes on cash and cash equivalents. |
| `totalCashFlowAfterEffectOfExchangeRateChanges` | Number / String | Total cash flow after the effect of exchange rate changes. |
| **Remaining Balance Sheet Fields** | Number / String | Includes balance sheet values such as Share Capital, Reserves, Borrowings, Assets, Liabilities, Investments, Cash & Cash Equivalents, Inventory, Trade Receivables, Trade Payables, and other balance sheet items. Returns `"Null"` when the corresponding value is unavailable. |


### Retrieval Logic

#### Step 1 - Retrieve Income Statement

Lookup:
ace_balancesheet_results_ind_as_format_data_merged_ace_financial

    Filter:
    1. Fincode
    2. nature = 'S'
    3. nature = 'C'

Sort by Date_End (Descending) and select latest record.

#### Step 2 - Retrieve Balance Sheet

Lookup:
ace_balancesheet_result_balancesheet

    Filter:
    1. FinCode
    2. nature = 'S' / 'C'

Sort by Date_End (Descending) and select latest record.

#### Step 3 - Capitaline Fallback

    Retrieve CapitalineCode using:
    1. FINCODE
    2. SCRIPCODE
    3. ISIN

If found:
Lookup capitaline_new_download
Sort by YearEnd (Descending)
Select latest record.

#### Step 4 - Retrieve Cash Flow

Cash flow information is retrieved in the following priority:

```
  1. ace_balancesheet_result_cashflow
  
  If unavailable,
  
  2. ace_financial_cf
```
   
Cash flow values populate:

 - Operating Cash Flow
 - Investing Cash Flow
 - Financing Cash Flow
 - Total Cash Flow
 - Exchange Rate Adjustment

#### Step 5 - Financial Year

The `financialYear` field is generated using the `nse_financial_year_1()` function.

The function:

  - Retrieves Year End information from ACE Balance Sheet.
  - Falls back to ACE Financial Cash Flow.
  - Falls back to Capitaline YearEnd.
  - Compares YearEnd with Date_End.
  - Handles financial years longer than 12 months (including 15-month reporting periods).
  - Returns the financial year in the format:
    ```
    01 Jul 2024 - 30 Jun 2025
    or
    01 Jan 2025 - 31 Dec 2025
    ```
    depending on the company's reporting cycle:
    
#### Step 6 - Populate FinancialResults

The API populates all quarterly, YTD, balance sheet, and cash flow fields from the selected latest records.

When both Standalone and Consolidated records are available:

```
FinancialResults =
[
   Standalone,
   Consolidated
]
```

If only one report exists, only that object is returned.

#### Step 7 - Calculate YTD

After data retrieval:

    ytd_calculations(fin_code, result)

is executed to populate all Year-To-Date (YTD) financial values.

#### Step 8 - Value Conversion

Before returning the response:

  - If data originates from `Capitaline`, numeric values are multiplied by `10,000,000`.
  - Otherwise (ACE data), numeric values are multiplied by `1,000,000`.
  - EPS values are returned without conversion.

### Null Handling

-   If an exception occurs, every field is returned as `"Null"`.
-   Database NULL values are returned as `"Null"`.
-   If cash flow is unavailable, fallback is attempted.
-   If financial year cannot be derived, `financialYear` is `"Null"`.

### Notes

-   Latest financial records are selected using Date_End.
-   Cash flow uses fallback logic.
-   Capitaline is used only when NSE data is unavailable.
-   YTD values are calculated after retrieval.
-   Numeric values are converted before returning.
-   Database NULL values are returned as `"Null"`.

