# Pledge Invocation

The `pledgeInvocation` object provides promoter share encumbrance details from both **BSE** and **NSE** sources.

The API attempts to retrieve BSE and NSE pledge data independently. If data is unavailable or an exception occurs, the respective exchange object returns `"Null"` for all fields.

## Response Structure

| Field | Type | Description |
|-------|------|-------------|
| `pledgeInvocation` | Object | Contains pledge and encumbrance details from BSE and NSE. |
| `pledgeInvocation.bse` | Object | Pledge data retrieved from BSE. |
| `pledgeInvocation.nse` | Object | Pledge data retrieved from NSE. |

### Pledge Fields

| Field | Type | Description |
|-------|------|-------------|
| `nameOfPromoterOrPAC` | String | Name of the promoter or person/group acting in concert (PAC). |
| `promoterHoldingNoOfShares` | Number / String | Number of shares held by the promoter. |
| `promoterHoldingPerOfTotalCapital` | Number / String | Promoter holding as a percentage of total share capital. |
| `promoterHoldingSharesAlreadyEncumberedNoOfShares` | Number / String | Number of promoter-held shares that are already encumbered. |
| `promoterHoldingSharesAlreadyEncumberedPerOfTotalCapital` | Number / String | Percentage of total share capital represented by already encumbered promoter shares. |
| `typeOfEventCreationReleaseInvocationOnEncumbrance` | String | Type of encumbrance event, such as Creation, Release, or Invocation. |
| `creationReleaseInvocationOnEncumbranceNoOfShares` | Number / String | Number of shares involved in the creation, release, or invocation event. |
| `creationReleaseInvocationOnEncumbrancePerOfTotalCapital` | Number / String | Percentage of total share capital involved in the event. |
| `dateOfEventCreationReleaseInvocationOnEncumbrance` | String | Date of the encumbrance event. BSE date is formatted as `YYYY-MM-DD`. |
| `nameOfEntityInWhoseFavourSharesEncumbered` | String | Name of the entity in whose favour the shares are encumbered. Returns `"Null"` if unavailable. |

## Retrieval Logic

### BSE Pledge Data

BSE pledge data is retrieved using the **ISIN**.

The ISIN is identified using the following lookup sequence:

1. Search `bse_new_security_list` using `SCRIPCODE`.
2. If ISIN is not found, search `new_mapping` using `SCRIPCODE`.
3. If ISIN is still unavailable and `PAN` is provided:
   - Search `bse_new_security_list` using `PAN`.
   - If not found, the code attempts the `new_mapping` lookup.
4. If no ISIN is available, `"Null"` is used.
5. The retrieved ISIN is passed to `pledgeDatas()`.

### `pledgeDatas()` Function

The `pledgeDatas()` function retrieves and prepares BSE pledge data.

The function performs the following operations:

1. Accepts **ISIN** and **Target Company Name** as input parameters.
2. Builds the filter dynamically based on the available input values.
3. Queries the `bse_pledge` table using:
   - `isin_number` when ISIN is provided.
   - `name_of_the_target_company` when the target company name is provided.
4. Retrieves the promoter holding, encumbered shares, event details, event date, and entity information.
5. Maps the database fields to the API response field names.
6. Converts empty source values to `"Null"`.
7. Verifies the availability of the corresponding security code from `bse_new_security_list` using the retrieved ISIN.
8. If matching pledge data is available, returns the mapped pledge information with HTTP status `200`.
9. If no matching record is found, returns `"No Record Found"` with status `500`.
10. If an exception occurs, the function returns the exception with status `500`.
11. The BSE event date is converted from `DD/MM/YYYY` to `YYYY-MM-DD` before being added to the API response.


### BSE Source

| Source Table | Lookup Column | Selected Data |
|--------------|---------------|---------------|
| `bse_pledge` | `isin_number` | Pledge and encumbrance details |
| `bse_new_security_list` | `isin` | Security code mapping |

The `bse_pledge` table provides the following source fields:

| Source Field | Response Field |
|--------------|----------------|
| `name_of_the_promoters` | `nameOfPromoterOrPAC` |
| `ph_in_company_no_of_shares` | `promoterHoldingNoOfShares` |
| `ph_in_company_per_of_equityshare` | `promoterHoldingPerOfTotalCapital` |
| `ph_in_encumbered_no_of_shares` | `promoterHoldingSharesAlreadyEncumberedNoOfShares` |
| `ph_in_encumbered_per_of_equityshare` | `promoterHoldingSharesAlreadyEncumberedPerOfTotalCapital` |
| `depe_types_of_event` | `typeOfEventCreationReleaseInvocationOnEncumbrance` |
| `depe_no_of_shares` | `creationReleaseInvocationOnEncumbranceNoOfShares` |
| `depe_per_of_equityshare` | `creationReleaseInvocationOnEncumbrancePerOfTotalCapital` |
| `depe_date_of_creation` | `dateOfEventCreationReleaseInvocationOnEncumbrance` |
| `depe_name_of_entity` | `nameOfEntityInWhoseFavourSharesEncumbered` |

The BSE event date is converted from `DD/MM/YYYY` to `YYYY-MM-DD`.

### NSE Pledge Data

NSE pledge data is retrieved using the `security_id`.

The `security_id` is obtained from `bse_new_security_list` using the provided PAN.

The retrieved security ID is passed to `nse_pledge_data()`.

### `nse_pledge_data()` Function

The `nse_pledge_data()` function retrieves and prepares the latest NSE pledge data.

The function performs the following operations:

1. Accepts the **security ID** as the input parameter.
2. Queries the `nse_pledge_new` table using the security ID.
3. Retrieves promoter holding, encumbered shares, event details, event date, entity information, and `broadcast_date_time`.
4. Sorts the retrieved records by `broadcast_date_time` in ascending order.
5. Selects the **latest available record** from the sorted records.
6. Maps the selected database record to the API response field names.
7. Returns `"Null"` values when no data is available or an exception occurs.
8. The NSE event date is returned using the available source date value.

### NSE Source

| Source Table | Lookup Column | Selected Record |
|--------------|---------------|-----------------|
| `nse_pledge_new` | `symbol` | Latest record based on `broadcast_date_time` |

When multiple NSE pledge records are available, the records are sorted by `broadcast_date_time` in ascending order, and the latest record is selected.

The following NSE source fields are mapped to the API response:

| Source Field | Response Field |
|--------------|----------------|
| `name_of_pro_or_pacs_with_him` | `nameOfPromoterOrPAC` |
| `ph_in_the_target_company_number` | `promoterHoldingNoOfShares` |
| `ph_in_the_target_company_per_of_total_sharecapita` | `promoterHoldingPerOfTotalCapital` |
| `ph_already_encumbered_numbe` | `promoterHoldingSharesAlreadyEncumberedNoOfShares` |
| `ph_already_encumbered_per_of_total_sharecapital` | `promoterHoldingSharesAlreadyEncumberedPerOfTotalCapital` |
| `dt_events_pertaining_to_encumbrance_type_of_event` | `typeOfEventCreationReleaseInvocationOnEncumbrance` |
| `dt_events_pertaining_to_encumbrance_number` | `creationReleaseInvocationOnEncumbranceNoOfShares` |
| `dt_events_pertaining_to_encumbrance_per_of_total_sharecapita` | `creationReleaseInvocationOnEncumbrancePerOfTotalCapital` |
| `dt_events_pertaining_to_encumbrance_date_of_even` | `dateOfEventCreationReleaseInvocationOnEncumbrance` |
| `dt_events_pertaining_encumbrance_name_entity_shares_encumbere` | `nameOfEntityInWhoseFavourSharesEncumbered` |

## Null Handling

- The `pledgeInvocation` object is initialized with separate `bse` and `nse` objects.
- If BSE pledge data is unavailable, all BSE pledge fields are returned as `"Null"`.
- If NSE pledge data is unavailable, all NSE pledge fields are returned as `"Null"`.
- Empty source values are converted to `"Null"` where applicable.
- If no matching pledge record is found, the corresponding exchange object returns `"Null"` values.
- BSE event dates are returned in `YYYY-MM-DD` format when valid.
- NSE date values are returned based on the available source value.

## Notes

- BSE and NSE pledge data are retrieved independently.
- BSE primarily uses **ISIN** for pledge data retrieval.
- NSE uses the **security ID** associated with the PAN.
- For NSE, the most recent record is selected based on `broadcast_date_time`.
- The API does not fail the complete response when pledge data for one exchange is unavailable; instead, the unavailable exchange returns `"Null"` values.
