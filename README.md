# BTP Job Scheduler Service

![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0.3-6BA539?logo=openapiinitiative&logoColor=white)
![Formats](https://img.shields.io/badge/formats-JSON%20%7C%20YAML-blue)
![Status](https://img.shields.io/badge/spec%20lint-passing-brightgreen)

OpenAPI metadata for the **SAP Job Scheduling Service REST API** (SAP BTP) — create, schedule, monitor, and secure jobs via HTTP endpoints or Cloud Foundry tasks.

This README documents a validation pass performed on `sap-btpjss-admin-v1.json` / `sap-btpjss-admin-v1.yaml`: the errors found, why each one matters, and exactly how it was fixed.

---

## Table of Contents

- [Errors found](#errors-found)
  - [1. Identical/ambiguous paths](#1-identicalambiguous-paths-error)
  - [2. `nullable` used without `type`](#2-nullable-used-without-type-error)
  - [3. Examples that don't match their declared `format`](#3-examples-that-dont-match-their-declared-format-warning)
  - [4. Required field missing from its own examples](#4-required-field-missing-from-its-own-examples-warning)
- [Before / after](#before--after)
- [Summary](#summary)

---

## Errors found

| # | Issue | Rule | Severity | Occurrences |
|---|-------|------|----------|:-----------:|
| 1 | `/jobs/{jobId}` and `/jobs/{name}` are identical path templates | `no-identical-paths` | 🔴 Error | 1 |
| 2 | `nullable: true` used without a sibling `type` | `nullable-type-sibling` | 🔴 Error | 4 |
| 3 | Example values didn't match their declared `format` (`date-time`, `uuid`) | `no-invalid-schema-examples` | 🟡 Warning | 12 |
| 4 | `endTime` marked `required` but absent from every request example | `no-invalid-media-type-examples` | 🟡 Warning | 4 |

<br>

### 1. Identical/ambiguous paths (Error)

**What was wrong:** the spec declared two separate path items —

```
PUT /jobs/{jobId}   (update an existing job by numeric ID)
PUT /jobs/{name}    (create/update a job by name)
```

The OpenAPI Specification explicitly states that path templates with the same segment structure but *different parameter names* count as the **same path** — `/jobs/{jobId}` and `/jobs/{name}` are indistinguishable to a router. This is why it's flagged as an error, not a style nit: most API gateways, codegen tools, and mock servers can only bind **one** operation to that URL shape. A client calling the "other" one is liable to get routed to the wrong operation entirely — commonly surfacing as a **404**.

**How it was fixed:** the two `PUT` operations were merged into a single operation on `/jobs/{jobId}`, with:
- one path parameter typed as `oneOf: [integer, string]` (numeric ID *or* job name), documented accordingly,
- one request body typed as `anyOf: [UpdateJobRequest, CreateJobRequest]` so either shape validates,
- one response set covering both outcomes (`200` update-by-ID, `200`/`201` configure-by-name),
- a merged description explaining the two behaviors.

No real endpoint URL changed — this only restructured the *documentation* to be unambiguous.

### 2. `nullable` used without `type` (Error)

**What was wrong:** `startTime` / `endTime` on `CreateJobRequest` and `UpdateJobRequest` used this pattern:

```json
"oneOf": [
  { "type": "string", "format": "date-time" },
  { "$ref": "#/components/schemas/DateTimeObject" },
  { "nullable": true }
]
```

OpenAPI 3.0 requires `type` to be present whenever `nullable` is used — a bare `{ "nullable": true }` schema is invalid and most tooling either errors out or silently ignores it.

**How it was fixed:** the bare `nullable` branch was removed, and `nullable: true` was attached directly to the schema that already had `type: string`:

```json
"oneOf": [
  { "type": "string", "format": "date-time", "nullable": true },
  { "$ref": "#/components/schemas/DateTimeObject" }
]
```

Same effective validation (string, object, or `null`) — now spec-compliant.

### 3. Examples that don't match their declared `format` (Warning)

**What was wrong:** several `example` values were tagged `format: date-time` or `format: uuid` but didn't actually satisfy that format:

| Field | Before | Problem |
|---|---|---|
| `createdAt` | `"1970-01-01 12:55:55"` | space instead of `T`, no timezone |
| `startTime` | `"2015-10-20 04:30:00"` | same |
| `nextRunAt`, `modifiedAt` | `"2017-08-11 10:00:00"` | same |
| `executionTimestamp`, `scheduleTimestamp`, `completionTimestamp` | `"2015-11-14T04:19:22"` | has `T`, still missing timezone offset |
| `runId` | `"56468DB7B133728EE10000000A61A0D8"` | valid 32 hex chars, but missing UUID dashes |

`format: date-time` requires full RFC 3339 (`T` separator **and** a timezone offset/`Z`).

**How it was fixed:** dates were reformatted to RFC 3339 UTC, e.g. `2015-10-20T04:30:00Z`; the `runId` example was re-grouped into standard `8-4-4-4-12` UUID form: `56468db7-b133-728e-e100-00000a61a0d8`.

### 4. Required field missing from its own examples (Warning)

**What was wrong:** `CreateJobRequest.required` lists `endTime` as mandatory, but none of the four documented request-body examples (`cronJob`, `oneTimeJob`, `createCronJob`, `updateExistingJob`) included it — so the spec's own examples didn't validate against its own schema.

**How it was fixed:** added `"endTime": null` to each of the four examples, consistent with `endTime` being required-but-nullable.

---

## Before / after

```diff
- ❌ 5 errors, 17 warnings   (JSON)
- ❌ 5 errors, 17 warnings   (YAML)
+ ✅ 0 errors, 0 warnings    (JSON)
+ ✅ 0 errors, 0 warnings    (YAML)
```

## Summary

| File | Errors before | Warnings before | Errors after | Warnings after |
|---|:---:|:---:|:---:|:---:|
| `sap-btpjss-admin-v1.json` | 5 | 17 | 0 | 0 |
| `sap-btpjss-admin-v1.yaml` | 5 | 17 | 0 | 0 |

Both files now pass `openapi-spec-validator` and the full Redocly recommended ruleset, and remain structurally identical to each other.
