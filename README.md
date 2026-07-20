# Django_Employee_Management
Response Object Description

The API returns market data under the result.data object.

Field	Type	Description
bseStatus	String	Current trading status of the company on the Bombay Stock Exchange (BSE). Possible values include Active, Suspended, Delisted, etc.
nseStatus	String	Current trading status of the company on the National Stock Exchange (NSE). Possible values include Active, Suspended, Delisted, etc.
EquityData	Object	Contains the latest equity share information such as face value, paid-up capital, issued capital, number of shares, and other equity-related details.
52WeekHL	Object	Contains the company's 52-week High and Low trading information for both BSE and NSE, including prices and the dates on which the high and low occurred.
MarketCapitalData	Object	Contains market capitalization details such as market capitalization value, free-float market capitalization (if available), and related valuation metrics.
FinancialResults	Array	Contains the latest published financial statements of the company. The array may include Standalone and/or Consolidated results with quarterly, half-yearly, or annual financial information such as revenue, profit, EPS, assets, liabilities, cash flow, and balance sheet details.
Detailed Description
bseStatus

Represents the listing status of the company on the Bombay Stock Exchange (BSE).

Possible values

Value	Meaning
Active	Company is actively listed and traded on BSE.
Suspended	Trading has been suspended by the exchange.
Delisted	Company has been removed from the exchange.
Null	Status information is unavailable.
nseStatus

Represents the listing status of the company on the National Stock Exchange (NSE).

Possible values

Value	Meaning
Active	Company is actively listed and traded on NSE.
Suspended	Trading has been suspended by the exchange.
Delisted	Company has been removed from the exchange.
Null	Status information is unavailable.
EquityData

Provides information related to the company's equity shares.

Typical information includes:

Face Value
Paid-up Equity Capital
Issued Capital
Authorized Capital
Number of Equity Shares
ISIN
Equity Share Statistics
52WeekHL

Provides the company's trading range during the last 52 weeks.

Typical fields include:

Field	Description
bseHigh	Highest price traded on BSE during the last 52 weeks.
bseHighDate	Date on which the BSE high occurred.
bseLow	Lowest price traded on BSE during the last 52 weeks.
bseLowDate	Date on which the BSE low occurred.
nseHigh	Highest price traded on NSE during the last 52 weeks.
nseHighDate	Date on which the NSE high occurred.
nseLow	Lowest price traded on NSE during the last 52 weeks.
nseLowDate	Date on which the NSE low occurred.
MarketCapitalData

Contains market valuation information.

Typical information includes:

Market Capitalization
Free Float Market Capitalization
Enterprise Value (if available)
Currency
Last Updated Date
FinancialResults

Contains the company's published financial performance.

The array can contain one or more reports such as:

Standalone Financial Results
Consolidated Financial Results

Each report may include:

Reporting Quarter
Nature of Report
Financial Year
Revenue / Net Sales
Other Income
EBITDA (if available)
Profit Before Tax (PBT)
Profit After Tax (PAT)
Earnings Per Share (EPS)
Interest
Depreciation & Amortisation
Cash Flow Details
Balance Sheet Information
Assets and Liabilities
Share Capital
Reserves & Surplus
Inventory
Trade Receivables
Borrowings
Cash & Cash Equivalents
