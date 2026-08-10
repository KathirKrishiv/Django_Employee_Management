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

The retrieved ISIN is passed to `pledgeDatas()`.

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
