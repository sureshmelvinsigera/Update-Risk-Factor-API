# Update Risk Factor API

#### Purpose
Update the risk factor for a flagged driver after an MVR (Motor Vehicle Report) correction has been verified. This API enables underwriting teams to recalculate and apply corrected risk classifications when a driver's MVR record has been updated or an error in the original report has been resolved.

#### Endpoint
`PUT /api/v1/drivers/risk-factor`

#### Consumers
- Underwriting Portal
- MVR Correction Workflow System
- Agent Dashboard

#### Dependencies
- Policy Management System (validates PolicyNumber and driver association)
- MVR Provider Integration (confirms MVR correction has been processed)
- Risk Engine (validates the new RiskFactor against rating tables)
- pc_driver table (source of driver records and current risk factor values)
- Rating_Table (defines valid risk factor ranges and tier mappings)

## Request

#### Body Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| PolicyNumber | String | Yes | The unique policy identifier. Expected format: `POL-YYYY-NNNNN` |
| DriverID | String | Yes | The unique identifier for the driver on the policy (from pc_driver table) |
| NewRiskFactor | Decimal | Yes | The corrected risk factor value. Must fall within the valid range defined in the Rating_Table (0.5 – 4.0) |
| CorrectionReason | String | Yes | Reason for the risk factor update (e.g., "MVR record corrected", "Violation removed", "Duplicate record resolved") |
| MVR_CorrectionDate | Date | Yes | The date the MVR correction was confirmed by the provider (YYYY-MM-DD) |

#### Example Request
```json
PUT /api/v1/drivers/risk-factor

{
  "PolicyNumber": "POL-2025-00482",
  "DriverID": "DRV-10234",
  "NewRiskFactor": 1.2,
  "CorrectionReason": "MVR record corrected — speeding violation removed after court dismissal",
  "MVR_CorrectionDate": "2026-02-20"
}
```

## Response

#### Fields

| Field | Type | Description |
|---|---|---|
| PolicyNumber | String | The policy that was updated |
| DriverID | String | The driver whose risk factor was updated |
| DriverName | String | Full name of the driver (returned for confirmation) |
| PreviousRiskFactor | Decimal | The risk factor value before the update |
| NewRiskFactor | Decimal | The corrected risk factor value now applied |
| PreviousRiskTier | String | The risk tier before the update (Low, Medium, High, Critical) |
| NewRiskTier | String | The recalculated risk tier after the update |
| PreviousPremium | Decimal | The total premium before the update (USD) |
| NewPremium | Decimal | The recalculated total premium after the update (USD) |
| UpdatedAt | DateTime | Timestamp of when the update was applied (ISO 8601) |
| UpdatedBy | String | The system or user that initiated the update |

#### Example Response (Success)
```json
{
  "PolicyNumber": "POL-2025-00482",
  "DriverID": "DRV-10234",
  "DriverName": "Jane Smith",
  "PreviousRiskFactor": 3.1,
  "NewRiskFactor": 1.2,
  "PreviousRiskTier": "High",
  "NewRiskTier": "Low",
  "PreviousPremium": 2890.00,
  "NewPremium": 1245.50,
  "UpdatedAt": "2026-02-27T14:32:00Z",
  "UpdatedBy": "underwriter@zurich.com"
}
```

## Error Handling

| Scenario | Error Code | Error Message |
|---|---|---|
| PolicyNumber is missing from the request | POLICY_NUMBER_MISSING | "PolicyNumber is required." |
| PolicyNumber not found in the system | POLICY_NOT_FOUND | "No policy found for the provided PolicyNumber." |
| DriverID is missing from the request | DRIVER_ID_MISSING | "DriverID is required." |
| DriverID does not exist on the specified policy | DRIVER_NOT_FOUND | "No driver found with the provided DriverID on this policy." |
| NewRiskFactor is outside the valid range (0.5 – 4.0) | RISK_FACTOR_OUT_OF_RANGE | "NewRiskFactor must be between 0.5 and 4.0 per the Rating_Table." |
| Driver's MVR_Status is not currently "Flagged" | DRIVER_NOT_FLAGGED | "Risk factor can only be updated for drivers with a Flagged MVR status." |
| MVR correction has not been confirmed by the provider | MVR_CORRECTION_NOT_VERIFIED | "MVR correction has not been verified by the provider. Update cannot proceed." |
| Policy is inactive (cancelled or expired) | POLICY_INACTIVE | "The policy associated with this PolicyNumber is no longer active." |
| CorrectionReason is missing or empty | CORRECTION_REASON_MISSING | "CorrectionReason is required to document the basis for this update." |

#### Example Error Response
```json
{
  "error": "NewRiskFactor must be between 0.5 and 4.0 per the Rating_Table.",
  "code": "RISK_FACTOR_OUT_OF_RANGE"
}
```

## Business Rules

1. **Valid Risk Factor Range:** `NewRiskFactor` must fall within **0.5 – 4.0** as defined in the Rating_Table. A value of 5.0 is invalid and must be rejected.
2. **Flagged Drivers Only:** The risk factor can only be updated for drivers whose current `MVR_Status` is **"Flagged"**. Drivers with a "Clear" status do not require correction.
3. **MVR Correction Must Be Verified:** The update is only permitted if the MVR provider has confirmed the correction. The system must validate against the MVR provider before applying changes.
4. **Risk Tier Recalculation:** When the risk factor is updated, the system must automatically recalculate the `RiskTier` based on the Rating_Table thresholds:
   - 0.5 – 1.5 → **Low**
   - 1.6 – 2.5 → **Medium**
   - 2.6 – 3.5 → **High**
   - 3.6 – 4.0 → **Critical**
5. **Premium Recalculation:** The `TotalPremium` must be recalculated based on the new RiskTier and returned in the response.
6. **Audit Trail Required:** Every risk factor update must be logged with the previous value, new value, CorrectionReason, timestamp, and the user/system that initiated the change.
7. **Active Policies Only:** Risk factor updates are only permitted on active policies. Cancelled or expired policies must return a `POLICY_INACTIVE` error.
8. **One Update Per Correction:** A single MVR correction event should result in only one risk factor update. Duplicate submissions for the same MVR_CorrectionDate and DriverID should be flagged.

## Risk Tier Mapping (Rating_Table Reference)

| Risk Factor Range | Risk Tier |
|---|---|
| 0.5 – 1.5 | Low |
| 1.6 – 2.5 | Medium |
| 2.6 – 3.5 | High |
| 3.6 – 4.0 | Critical |

## Assumptions & Constraints

- This endpoint updates the risk factor for a **single driver on a single policy** per request. Batch updates are not supported.
- The MVR correction must be confirmed by the third-party provider **before** this API is called. The system does not trigger MVR corrections — it only acts on confirmed ones.
- Risk factor changes take effect immediately upon successful update.
- This is a **write operation** — appropriate authorization is required. Only users with underwriting or admin roles should have access.
- The `pc_driver` table is the system of record for driver data and will be updated upon a successful request.

## Open Questions

| # | Question | Status | Owner |
|---|---|---|---|
| 1 | Should the API support bulk updates if multiple drivers on the same policy are affected? | Open | BA / Product |
| 2 | Is there a cooling-off period before a second risk factor update can be applied to the same driver? | Open | BA / Underwriting |
| 3 | Should the premium recalculation be synchronous (in the response) or handled asynchronously? | Open | Dev / Architecture |
| 4 | What role-based access controls are required for this endpoint? | Open | Security / Dev |
| 5 | Should a notification be sent to the policyholder when their premium changes due to an MVR correction? | Open | BA / Product |

## Revision History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-02-27 | BA Team | Initial draft based on Zurich lab exercise (Slide #30) |

## Notes
- Authentication, HTTP status codes, and performance requirements are owned by the development team.
- This document defines the business requirements only. Technical implementation details (e.g., OAuth 2.0, response time SLAs) are documented separately.
- Field names and valid ranges reference the **pc_driver** and **Rating_Table** tabs in the workbook.
- Reference: Lab Exercise — Slide #30
