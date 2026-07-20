# Response Object Description

The API returns market data under the `result.data` object.

  ----------------------------------------------------------------------------
  Field                   Type             Description
  ----------------------- ---------------- -----------------------------------
  **bseStatus**           String           Current trading status of the
                                           company on the **Bombay Stock
                                           Exchange (BSE)**. Possible values
                                           include `Active`, `Suspended`,
                                           `Delisted`, etc.

  **nseStatus**           String           Current trading status of the
                                           company on the **National Stock
                                           Exchange (NSE)**. Possible values
                                           include `Active`, `Suspended`,
                                           `Delisted`, etc.

  **EquityData**          Object           Contains the latest equity share
                                           information such as face value,
                                           paid-up capital, issued capital,
                                           number of shares, and other
                                           equity-related details.

  **52WeekHL**            Object           Contains the company's **52-week
                                           High and Low** trading information
                                           for both BSE and NSE, including
                                           prices and dates.

  **MarketCapitalData**   Object           Contains market capitalization
                                           details such as market
                                           capitalization value, free-float
                                           market capitalization (if
                                           available), and related valuation
                                           metrics.

  **FinancialResults**    Array            Contains the latest published
                                           Standalone and/or Consolidated
                                           financial statements including
                                           revenue, PAT, EPS, assets,
                                           liabilities, and cash flow
                                           information.
  ----------------------------------------------------------------------------

------------------------------------------------------------------------

## bseStatus

Represents the listing status of the company on the Bombay Stock
Exchange (BSE).

  Value       Meaning
  ----------- -----------------------------------------------
  Active      Company is actively listed and traded on BSE.
  Suspended   Trading has been suspended.
  Delisted    Company has been delisted.
  Null        Status unavailable.

------------------------------------------------------------------------

## nseStatus

Represents the listing status of the company on the National Stock
Exchange (NSE).

  Value       Meaning
  ----------- -----------------------------------------------
  Active      Company is actively listed and traded on NSE.
  Suspended   Trading has been suspended.
  Delisted    Company has been delisted.
  Null        Status unavailable.

------------------------------------------------------------------------

## EquityData

Typical information includes:

-   Face Value
-   Paid-up Equity Capital
-   Issued Capital
-   Authorized Capital
-   Number of Equity Shares
-   ISIN
-   Equity Share Statistics

------------------------------------------------------------------------

## 52WeekHL

Typical fields include:

  Field         Description
  ------------- -----------------------------------------
  bseHigh       Highest BSE price in the last 52 weeks.
  bseHighDate   Date of the BSE high.
  bseLow        Lowest BSE price in the last 52 weeks.
  bseLowDate    Date of the BSE low.
  nseHigh       Highest NSE price in the last 52 weeks.
  nseHighDate   Date of the NSE high.
  nseLow        Lowest NSE price in the last 52 weeks.
  nseLowDate    Date of the NSE low.

------------------------------------------------------------------------

## MarketCapitalData

Contains:

-   Market Capitalization
-   Free Float Market Capitalization
-   Enterprise Value (if available)
-   Currency
-   Last Updated Date

------------------------------------------------------------------------

## FinancialResults

Each object may contain:

-   Reporting Quarter
-   Nature of Report (Standalone/Consolidated)
-   Financial Year
-   Revenue / Net Sales
-   Other Income
-   Interest
-   Depreciation & Amortisation
-   Profit Before Tax (PBT)
-   Profit After Tax (PAT)
-   Earnings Per Share (EPS)
-   Cash Flow
-   Assets & Liabilities
-   Share Capital
-   Reserves & Surplus
-   Inventory
-   Trade Receivables
-   Borrowings
-   Cash & Cash Equivalents

------------------------------------------------------------------------

## Example

``` json
{
  "bseStatus": "Active",
  "nseStatus": "Active",
  "EquityData": {},
  "52WeekHL": {},
  "MarketCapitalData": {},
  "FinancialResults": [
    {
      "natureOfReport": "Standalone",
      "financialYear": "2024-25"
    }
  ]
}
```
