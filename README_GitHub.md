# Company Information

The following fields are returned under the `result.data` object.

**Source Table:** `new_mapping`\
**Lookup Fields:** `PAN`, `priority`

  ----------------------------------------------------------------------------------------------
  | Field              | Type      |   Description                |   Source |
  |------------------- |------------| ----------------------------- |------------------------------|
  |`nameofTheCompany`|   String   |    Official name of the company. | `new_mapping.COMPNAME` |

  |`scripCode`   |       String  |     BSE Scrip Code.     |          `new_mapping.SCRIPCODE`|

  |`pan`    |            String  |     PAN supplied in the API    |   Request Parameter|
                                    request.

  |`symbol`    |         String     |  Trading symbol. Falls back to |`new_mapping.SYMBOL`|
                                    `ace_company_master.symbol`
                                    if unavailable.|

  |`isin` |              String    |   ISIN of the company.     |     `new_mapping.ISIN`|

  |`listedWith`  |       String    |   Exchange(s) where the company 
                                      Derived from
                                    is listed (`BSE`, `NSE`,      `new_mapping.Bse_sublisting`
                                    `BSE & NSE`, `Null`).   |      and
                                                                  `new_mapping.Nse_sublisting` |
  ----------------------------------------------------------------------------------------------
---

### bseStatus

  |Property         | Value|
  |----------------- |---------------------------------------------|
  |Type|              String|
  |Description  |     Current BSE listing status of the security.|
  |Source Table |     `ace_company_master`|
  |Lookup Column |    `scripcode`|
  |Returned Column |  `bse_sublisting`|
  |Possible Values|   `Active`, `Suspended`, `Delisted`, `Null`|


------------------------------------------------------------------------

### nseStatus

  Property          Value
  ----------------- ---------------------------------------------
  Type              String
  Description       Current NSE listing status of the security.
  Source Table      `ace_company_master`
  Lookup Column     `fincode`
  Returned Column   `nse_sublisting`
  Possible Values   `Active`, `Suspended`, `Delisted`, `Null`


------------------------------------------------------------------------
---

### `EquityData`
- **Type:** `object`
- **Description:** The EquityData object contains the latest available equity market information for the company. The data is retrieved from the ace_equity table using the company's FINCODE.
  If multiple equity records exist, the API sorts them by `price_date` in ascending order and returns the most recent record.


**Source:**
|Source Table	|Lookup Column	|Sort Column	  |           Selected Record|
|-------------|-------------- |------------    |         -----------------|
|ace_equity	  |  fincode	    | price_date (Ascending)	| Latest (price_date)|

**Fields:**
`Field`	    Type	Description	Source Column
`clsPric`	Number / String	Latest closing market price of the equity. Returns "Null" if unavailable.	price
`stkExchange`	String	Stock exchange on which the latest equity price was recorded (BSE/NSE).	stk_exchange
`tradeDate`	Date	Trade date corresponding to the latest closing price.	price_date
`pe`	Number / String	Trailing Twelve Months (TTM) Price-to-Earnings ratio. Returns "Null" if unavailable.	ttmpe
`faceValue`	Number / String	Face value of the equity share.	fv
`faceValueDate`	Date	Date corresponding to the face value information.	price_date

---
## 52WeekHL

**Description:**
The `52WeekHL` object contains the company's **52-week High and Low prices** for both the **Bombay Stock Exchange (BSE)** and the **National Stock Exchange (NSE)**.

The data is retrieved from the `ace_52whl` table.

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
Lookup ace_52whl using:

1. scripcode
2. If no record is found, lookup using symbol
```


#### Case 2 - Scrip Code Not Available

```
Lookup ace_52whl using:

1. fincode
2. If no record is found, lookup using symbol
```

If a record is found:

- `scripCode` is updated from `ace_52whl.scripcode`.
- `symbol` is updated from `ace_52whl.symbol`.
- `bseStatus` is refreshed using the retrieved `scripCode`.

### Null Handling

If no matching record is found or an exception occurs, the API returns Null:


### Notes

- All dates are returned in **YYYY-MM-DD** format.
- Database `NULL` values are returned as the string `"Null"`.
- When the `scripCode` is unavailable, the API automatically attempts to retrieve the data using `fincode`, and then `symbol`.
- If a valid `scripCode` is obtained during the fallback lookup, the API updates the response with the new `scripCode`, `symbol`, and corresponding `bseStatus`.

---

# MarketCapitalData

The `MarketCapitalData` object provides the company's latest **Market Capitalization** details based on the most recent equity record available in the `ace_equity` table.

The data is retrieved using the company's **FINCODE**.

---

## Source

| Source Table | Lookup Column |
|--------------|---------------|
| `ace_equity` | `fincode` |

---

## Response Structure

| Field | Type | Description | Source Column |
|------|------|-------------|---------------|
| `marketCapValue` | Number / String | Latest market capitalization value (in Crores). | `mcap` |
| `stkExchange` | String | Stock exchange associated with the market capitalization record. | `stk_exchange` |
| `tradeDate` | Date | Date on which the market capitalization value was recorded. | `price_date` |

---

# Retrieval Logic

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

---

# Exchange Logic

The `MarketCapitalData` object is populated only when the stock exchange obtained from `EquityData.stkExchange` is either:

- `BSE`
- `NSE`

If the exchange is unavailable or contains any other value, the API returns `"Null"` for all fields.

---

# Null Handling

If the market capitalization data is unavailable or an exception occurs while retrieving the data, the API returns Null:

---

## Notes

- Market capitalization is retrieved from the `ace_equity.mcap` column.
- The value is converted from its stored unit into **Crores** by dividing it by **10,000,000**.
- The result is rounded to **2 decimal places**.
- The `tradeDate` is derived from `price_date` and returned in **YYYY-MM-DD** format.
- Database `NULL` values are returned as the string `"Null"`.
