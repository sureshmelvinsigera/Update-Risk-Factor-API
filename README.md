# Get Driver Details API

#### Purpose

Retrieve driver details and risk tier for a given policy to support underwriting review and MVR (Motor Vehicle Report) validation.

This API serves as the primary lookup for underwriters who need to verify a driver's identity, risk classification, and MVR status before making decisions on policy renewals, endorsements, or claims. It does not modify any data — it is strictly a read operation.

#### Endpoint

`GET /api/v1/drivers/details`

This is a GET request because it retrieves data without creating or changing anything. The parameters are passed in the query string, not in a request body. If you see a JSON body paired with a GET, that is a red flag — GET requests should never carry a body.

#### Source System

PolicyCenter (pc_driver, pc_policy)

The source system is where the data lives. In this case, PolicyCenter is the system of record for all policy and driver information. The response fields are pulled from two specific tables: pc_driver holds driver-level data like name, MVR status, and risk factor, while pc_policy holds policy-level data like total premium and effective date. Knowing the source system matters because it tells the development team where to query, and it tells the BA which data dictionary to reference when validating field names and types.

#### Target System

Requesting application (Underwriting Portal)

The target system is the application that calls the API and consumes the response. Here, the Underwriting Portal is the primary consumer — it displays the returned data to the underwriter. Identifying the target system upfront prevents confusion later. If a developer asks "who is calling this?", the answer is already documented.

#### Trigger

Underwriter requests driver details for a specific policy.

The trigger describes what event causes this API to be called. It is not a scheduled job or a system-to-system batch process — it is a human action. An underwriter opens a policy in the portal, and the portal calls this API to populate the driver details screen. Documenting the trigger helps everyone understand when and why this API gets called, which matters for estimating volume, load, and priority.

#### Consumers

- Underwriting Portal
- Claims Management System
- Agent Dashboard

While the Underwriting Portal is the primary consumer, other systems may also call this API. Listing all known consumers helps the development team understand who will be affected by changes, downtime, or breaking modifications to the response format.

#### Dependencies

- PolicyCenter (source of truth for PolicyNumber, driver records, and policy data)
- MVR Provider Integration (third-party service for motor vehicle report data)
- Risk Engine / Rating_Table (maps Risk_Factor values to RiskTier classification)

Dependencies are the systems and services that must be available for this API to function correctly. If PolicyCenter is down, the API cannot return any data. If the MVR provider is unreachable, the API can still return driver details but cannot include MVR status — which is why the graceful degradation section exists further down. The Rating_Table is an internal lookup that converts a numeric Risk_Factor into a human-readable RiskTier. It is not a separate service, but it is a dependency because if the table is missing or misconfigured, the RiskTier in the response will be wrong.

---

## Request

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| PolicyNumber | String | Yes | — | The unique policy identifier. Expected format: `ST-XX-YYYY-NNNN` (e.g., IL-PA-2024-0312) |
| State | String | No | — | Two-letter state code. If provided, filters results to the specified state. |
| EffectiveDate | Date | No | — | Date in YYYY-MM-DD format. If provided, returns the policy version active on this date. |
| IncludeMVR | Boolean | No | `true` | Whether to include MVR status. Set to `false` to exclude MVR_Status from the response. |

PolicyNumber is the only required parameter. It uniquely identifies a policy in PolicyCenter and follows a structured format: state code, line of business, year, and sequence number. The format itself is a business rule — if the incoming value does not match ST-XX-YYYY-NNNN, the request should be rejected before it ever reaches the database.

State and EffectiveDate are optional filters. State narrows the lookup when the same policy number pattern could theoretically appear across states. EffectiveDate is useful when a policy has been renewed or endorsed multiple times — it tells the API which version of the policy to return, rather than always defaulting to the most recent one.

IncludeMVR controls whether the response includes the MVR_Status field. Defaulting to true means most callers get MVR data without having to ask for it. Setting it to false is useful when the caller only needs driver and premium data and wants to avoid the latency that comes with querying the third-party MVR provider.

#### Example Requests

```
GET /api/v1/drivers/details?PolicyNumber=IL-PA-2024-0312
```

```
GET /api/v1/drivers/details?PolicyNumber=IL-PA-2024-0312&State=IL&EffectiveDate=2024-01-15&IncludeMVR=true
```

---

## Response

#### Fields

| Field | Type | Always Returned | Possible Values | Description |
|---|---|---|---|---|
| PolicyNumber | String | Yes | — | Echo of the requested PolicyNumber for confirmation |
| DriverName | String | Yes | — | Full name of the primary driver on the policy (from pc_driver) |
| TotalPremium | Decimal | Yes | — | Total premium amount in USD (from pc_policy.TotalPremium) |
| RiskTier | String | Yes | Low, Medium, High, Critical | Risk classification derived via Rating_Table lookup on Risk_Factor |
| MVR_Status | String | When IncludeMVR = true | Clear, Flagged | Motor Vehicle Report status (from pc_driver.MVR_Status) |
| Status | String | Yes | Active, Expired, Cancelled | Current policy status |
| EffectiveDate | Date | Yes | — | Policy effective date in ISO 8601 format (YYYY-MM-DD) |

The response echoes PolicyNumber back to the caller. This is a deliberate design choice — it allows the consuming application to confirm that the response matches the request without relying on the order of API calls. It is especially useful when multiple requests are fired in parallel.

RiskTier deserves special attention. It is not stored directly in the database. Instead, the system reads the numeric Risk_Factor from pc_driver, looks it up against the Rating_Table, and returns the corresponding tier. This means RiskTier is a derived field — if the Rating_Table thresholds change, the same Risk_Factor value could map to a different tier.

MVR_Status is conditionally returned. When IncludeMVR is true (the default), the field is present. When false, it is omitted entirely — not set to null, not set to an empty string, but absent from the response. This distinction matters for the consuming application's parsing logic.

#### Example Response (Success)

```json
{
  "PolicyNumber": "IL-PA-2024-0312",
  "DriverName": "James Smith",
  "TotalPremium": 4536.00,
  "RiskTier": "High",
  "MVR_Status": "Flagged",
  "Status": "Active",
  "EffectiveDate": "2024-01-15"
}
```

#### Example Response (IncludeMVR = false)

```json
{
  "PolicyNumber": "IL-PA-2024-0312",
  "DriverName": "James Smith",
  "TotalPremium": 4536.00,
  "RiskTier": "High",
  "Status": "Active",
  "EffectiveDate": "2024-01-15"
}
```

---

## Error Handling

| Scenario | Error Code | Error Message |
|---|---|---|
| PolicyNumber parameter is missing from the request | POLICY_NUMBER_MISSING | "PolicyNumber is required." |
| PolicyNumber is present but empty | POLICY_NUMBER_EMPTY | "PolicyNumber must not be empty." |
| PolicyNumber does not match expected format | POLICY_NUMBER_INVALID | "Invalid PolicyNumber format. Expected format: ST-XX-YYYY-NNNN." |
| PolicyNumber is valid but no matching policy exists | POLICY_NOT_FOUND | "No policy found for the provided PolicyNumber." |
| Policy exists but status is Expired | POLICY_EXPIRED | "Cannot retrieve details for expired policies." |
| Policy exists but status is Cancelled | POLICY_CANCELLED | "Cannot retrieve details for cancelled policies." |
| MVR provider is temporarily unavailable | MVR_SERVICE_UNAVAILABLE | "MVR data is temporarily unavailable. Driver details returned without MVR status." |

Each error scenario is specific and distinguishable. A missing parameter is different from an empty parameter is different from an invalid format. The consuming application should be able to read the error code and know exactly what went wrong without parsing the message text.

The expired and cancelled policy errors are business rule errors, not technical errors. The policy exists in the database — it simply is not in a state where driver details should be disclosed. This was an open question that has been resolved: the decision is to return a hard error, not partial data.

#### Example Error Response

```json
{
  "error": "No policy found for the provided PolicyNumber.",
  "code": "POLICY_NOT_FOUND"
}
```

#### Graceful Degradation

If the MVR provider is unavailable and `IncludeMVR=true`, the API should return all other fields successfully and omit `MVR_Status`, along with a warning message indicating MVR data was unavailable. This prevents a full request failure due to a third-party dependency.

This is an important design decision. The MVR provider is external and outside the organisation's control. If the API returned a hard failure every time the provider was slow or unreachable, the Underwriting Portal would be unusable during those windows. Graceful degradation means the underwriter still gets the driver's name, premium, risk tier, and policy status — they just do not get the MVR status until the provider is back online.

---

## Business Rules

1. `PolicyNumber` is required for every request.
2. `PolicyNumber` must match the format `ST-XX-YYYY-NNNN` (State-LineOfBusiness-Year-SequenceNumber).
3. If `State` is provided, the response is filtered to that state. If omitted, no state filter is applied.
4. If `EffectiveDate` is provided, the API returns the policy version that was active on that date. If omitted, the current/most recent version is returned.
5. If `IncludeMVR` is omitted, it defaults to `true` and `MVR_Status` is included in the response.
6. If `IncludeMVR` is set to `false`, the `MVR_Status` field is excluded from the response.
7. `RiskTier` is derived from the `Risk_Factor` value in pc_driver via a Rating_Table lookup. It is not stored directly — it is calculated at response time.
8. `MVR_Status` must be one of: Clear or Flagged.
9. Only the primary driver on the policy is returned. Multi-driver support is out of scope for this endpoint.
10. `TotalPremium` reflects the most recently calculated premium and may not account for pending endorsements.
11. If a policy status is Expired or Cancelled, the API must not return driver details — it must return a business rule error.

---

## Assumptions and Constraints

- This endpoint returns data for a single policy lookup only. Batch lookups are not supported.
- The PolicyNumber format is validated before querying downstream systems.
- MVR data is sourced from a third-party provider and may have a latency of up to 24 hours from the time of the most recent driving record update.
- This API is read-only — it does not create, update, or delete any policy or driver records.
- Field names and valid ranges reference the pc_driver and Rating_Table tabs in the workbook.

---

## Open Questions

| # | Question | Status | Owner |
|---|---|---|---|
| 1 | Should the API support retrieving details for secondary/additional drivers on a policy? | Open | BA / Product |
| 2 | What is the expected SLA for the MVR provider integration? | Open | Dev / Architecture |
| 3 | Should cancelled/expired policies return partial data or a hard error? | Resolved — Hard error | BA |
| 4 | Is there a need for pagination if multi-driver support is added in the future? | Open | BA / Dev |

---

## Notes

- Authentication, HTTP status codes, and performance requirements are owned by the development team.
- This document defines the business requirements only. Technical implementation details (e.g., OAuth 2.0, response time SLAs) are documented separately.
- Reference: Guided Walkthrough — Slide #29
