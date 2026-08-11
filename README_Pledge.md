## Pledge Invocation

The `pledgeInvocation` object provides promoter share encumbrance details from both **BSE** and **NSE** sources.

The API attempts to retrieve BSE and NSE pledge data independently. If data is unavailable or an exception occurs, the respective exchange object returns `"Null"` for all fields.

### Response Structure

| Field | Type | Description |
|-------|------|-------------|
| `pledgeInvocation` | Object | Contains pledge and encumbrance details from BSE and NSE. |
| `pledgeInvocation.bse` | Object | Pledge data retrieved from BSE. |
| `pledgeInvocation.nse` | Object | Pledge data retrieved from NSE. |

### Pledge Result Fields for BSE & NSE

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

### Retrieval Logic

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


#### `pledgeDatas()` Function:

Retrieves and prepares BSE pledge data.

1. Accepts **ISIN** and **Target Company Name**.
2. Filters `bse_pledge` using the available input.
3. Retrieves pledge, promoter holding, encumbrance, and event details.
4. Maps database fields to API response fields.
5. Converts empty values to `"Null"`.
6. Validates the ISIN against `bse_new_security_list`.
7. Returns the mapped data if a record is found.
8. Returns `"No Record Found"` if no data is available.
9. Returns `"Null"` values when an exception occurs.
10. Converts the BSE event date to `YYYY-MM-DD` format.


#### BSE Source

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

#### `nse_pledge_data()` Function:

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

#### NSE Source

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

### Null Handling

- The `pledgeInvocation` object is initialized with separate `bse` and `nse` objects.
- If BSE pledge data is unavailable, all BSE pledge fields are returned as `"Null"`.
- If NSE pledge data is unavailable, all NSE pledge fields are returned as `"Null"`.
- Empty source values are converted to `"Null"` where applicable.
- If no matching pledge record is found, the corresponding exchange object returns `"Null"` values.
- BSE event dates are returned in `YYYY-MM-DD` format when valid.
- NSE date values are returned based on the available source value.

### Notes

- BSE and NSE pledge data are retrieved independently.
- BSE primarily uses **ISIN** for pledge data retrieval.
- NSE uses the **security ID** associated with the PAN.
- For NSE, the most recent record is selected based on `broadcast_date_time`.
- The API does not fail the complete response when pledge data for one exchange is unavailable; instead, the unavailable exchange returns `"Null"` values.

## ShareHoldingPattern

The `shareHoldingPattern` object provides the latest shareholding details for the company. The API returns the **latest two quarters** based on the available `date_end` values from the `ace_shp` table.

### Retrieval Logic

1. Retrieve shareholding records from `ace_shp` using `FINCODE`.
2. Sort the records by `date_end` in descending order.
3. Select the latest **two quarters**.
4. Convert `date_end` into `Mon YYYY` format for the `quarter` field.
5. Map the database values to the API response structure.
6. Convert unavailable or `NULL` values to `"Null"`.
7. Return the two latest quarterly records as a list.
8. If no records are available or an exception occurs, the corresponding fields are returned as `"Null"`.

### Source

| Source Table | Lookup Column | Sort Column | Selected Records |
|--------------|---------------|-------------|------------------|
| `ace_shp` | `FINCODE` | `date_end` (Descending) | Latest 2 quarters |

### Response Structure

| Field | Description |
|-------|-------------|
| `quarter` | Reporting quarter in `Mon YYYY` format. |
| `totalPromoterGroup` | Promoter group shareholding details. |
| `totalPromoterGroup.tpgTotals` | Total promoter group shareholding and encumbrance percentage. |
| `totalPromoterGroup.tpgIndian` | Indian promoter shareholding details. |
| `totalPromoterGroup.tpgForeign` | Foreign promoter shareholding details. |
| `public` | Public shareholding details. |
| `public.publicTotal` | Total public shareholding. |
| `public.totalInstitution` | Institutional public shareholding. |
| `public.totalNonInstitution` | Non-institutional public shareholding. |
| `public.governmentTotal` | Government shareholding. |
| `nonPromoterNonPublic` | Non-promoter and non-public shareholding details. |
| `nonPromoterNonPublic.totalCustodians` | Custodian/DR shareholding details. |
| `nonPromoterNonPublic.nonpromoternonpublic` | Non-promoter, non-public shareholding details. |
| `grandTotal` | Total number of shares and percentage of total share capital. |

### Shareholding Fields

| Field | Description |
|-------|-------------|
| `totalNoSharesHeld` | Total number of shares held. |
| `shareholdingAsAPerOfTotalNoOfShares` | Shareholding percentage of total shares. |
| `totalPercentEncumbered` | Percentage of promoter holding that is encumbered. |

### Data Mapping

The following source fields from `ace_shp` are mapped to the API response:

| Source Field | Response Field |
|--------------|----------------|
| `nsFtotalpromoter` | `totalPromoterGroup.tpgTotals.totalNoSharesHeld` |
| `tpFtotalpromoter` | `totalPromoterGroup.tpgTotals.shareholdingAsAPerOfTotalNoOfShares` |
| `pshGrandTotal` | `totalPromoterGroup.tpgTotals.totalPercentEncumbered` |
| `nsINDSubtotal` | `totalPromoterGroup.tpgIndian.totalNoSharesHeld` |
| `tpINDSubtotal` | `totalPromoterGroup.tpgIndian.shareholdingAsAPerOfTotalNoOfShares` |
| `nsFSubtotal` | `totalPromoterGroup.tpgForeign.totalNoSharesHeld` |
| `tpFSubtotal` | `totalPromoterGroup.tpgForeign.shareholdingAsAPerOfTotalNoOfShares` |
| `nsTotalpublic` | `public.publicTotal.totalNoSharesHeld` |
| `tpTotalpublic` | `public.publicTotal.shareholdingAsAPerOfTotalNoOfShares` |
| `nsINSubtotal` | `public.totalInstitution.totalNoSharesHeld` |
| `tpINSubtotal` | `public.totalInstitution.shareholdingAsAPerOfTotalNoOfShares` |
| `nsNINSubtotal` | `public.totalNonInstitution.totalNoSharesHeld` |
| `tpNINSubtotal` | `public.totalNonInstitution.shareholdingAsAPerOfTotalNoOfShares` |
| `nsINcgovt` | `public.governmentTotal.totalNoSharesHeld` |
| `tpINcgovt` | `public.governmentTotal.shareholdingAsAPerOfTotalNoOfShares` |
| `nsCustodianDRs` | `nonPromoterNonPublic.totalCustodians.totalNoSharesHeld` |
| `tpCustodianDRs` | `nonPromoterNonPublic.totalCustodians.shareholdingAsAPerOfTotalNoOfShares` |
| `nsCustodianDRs` | `nonPromoterNonPublic.nonpromoternonpublic.totalNoSharesHeld` |
| `tpCustodianDRs` | `nonPromoterNonPublic.nonpromoternonpublic.shareholdingAsAPerOfTotalNoOfShares` |
| `nsGrandTotal` | `grandTotal.totalNoSharesHeld` |
| `tpGrandTotal` | `grandTotal.shareholdingAsAPerOfTotalNoOfShares` |

### Null and Error Handling

- If a source value is `NULL`, the API returns `"Null"`.
- If no `ace_shp` records are found, the function returns a default response with `"Null"` values.
- If an exception occurs, the API returns the default shareholding structure.
- `shareHoldingPattern` is always returned as a **list** in the final API response.
- When data is available, the list contains the **latest two quarters**.
