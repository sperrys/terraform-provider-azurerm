# `azurerm_api_connection`: add `kind` property (V1/V2 support)

## Description

Adds the `kind` property to the `azurerm_api_connection` resource and data source,
enabling Logic App Standard customers to create V2 API connections.

### Problem

`azurerm_api_connection` was always creating connections with `kind = "V1"` with no
way to specify `V2`. Logic App Standard workflows **require** `kind = "V2"` because
only V2 connections expose the `connectionRuntimeUrl` property that Logic App Standard
uses at runtime to connect to managed APIs.

### Root cause

The `kind` field was present in the `2015-08-01-preview` swagger spec but was
accidentally omitted when the stable `2016-06-01` spec was rewritten from scratch in
March 2018 (azure-rest-api-specs PR #2761). The Azure backend has always accepted and
persisted both `V1` and `V2` — the omission was purely in the spec, not in the API
itself.

**Live API testing confirmed:**
- `PUT` with `kind: "V2"` → HTTP 200, value persisted
- `GET` response returns `"kind": "V2"` and includes `connectionRuntimeUrl`
- In-place kind change → HTTP 400 `"kind cannot be changed from V1 to V2"`
  (which is why `ForceNew: true` is set on the schema field)
- Only `V1` and `V2` are valid — any other value returns a typed enum error

### Changes

#### `azurerm_api_connection` (resource)
- Added `kind` — `Optional`, `Default: "V1"`, `ForceNew: true`,
  `ValidateFunc: StringInSlice(["V1", "V2"])`
- Create: passes `Kind` to the ARM model
- Read: reads `Kind` back from the API response

#### `azurerm_api_connection` (data source)
- Added `kind` as a `Computed` attribute

#### Tests
- Added `TestAccApiConnection_kindV2` acceptance test which:
  - Creates a connection with `kind = "V2"`
  - Asserts the value is returned as `"V2"` from the API
  - Runs an import step to verify `kind` survives a state import round-trip

#### Documentation
- `website/docs/r/api_connection.html.markdown` — added `kind` argument reference
  and a V2 example usage block for Logic App Standard
- `website/docs/d/api_connection.html.markdown` — added `kind` to attributes reference

---

## SDK Bump: `go-azure-sdk` → `v0.20260819.1085932`

This PR bumps `go-azure-sdk` from `v0.20260709.1191450` to `v0.20260819.1085932`.
This version includes the `Kind` field on `ApiConnectionDefinition` generated from
the Pandora workaround (hashicorp/pandora#5543).

### Collateral fixes from the SDK bump

The bump renamed three types in `insights/2021-05-01-preview/diagnosticsettings`:

| Old name | New name |
|---|---|
| `LogSettings` | `DiagnosticsLogSettings` |
| `MetricSettings` | `DiagnosticsMetricSettings` |
| `RetentionPolicy` | `MicrosoftCommonRetentionPolicy` |

`DiagnosticSettingsCategoryListOperationResponse.Model` also changed from a wrapper
struct with a `.Value` field to a direct `*[]DiagnosticSettingsCategoryResource` slice.

Affected files:
- `internal/services/monitor/monitor_diagnostic_setting_resource.go`
- `internal/services/monitor/monitor_diagnostic_categories_data_source.go`

---

## Testing

### `azurerm_api_connection`
```
TestAccApiConnection_kindV2           — PASS (new test)
TestAccApiConnection_basic            — PASS
TestAccApiConnection_requiresImport   — PASS
TestAccApiConnection_complete         — PASS
TestAccApiConnectionDataSource_basic  — PASS
```

### Monitor diagnostic (collateral SDK bump fixes)
```
TestAccMonitorDiagnosticSetting_enabledLogs               — PASS (460s)
TestAccMonitorDiagnosticSetting_enabledLogsMix             — PASS (594s)
TestAccMonitorDiagnosticSetting_enabledLogsCategoryGroup   — PASS (455s)
TestAccMonitorDiagnosticSetting_updateEnabledMetric        — PASS (304s)
TestAccDataSourceMonitorDiagnosticCategories_appService    — PASS (239s)
TestAccDataSourceMonitorDiagnosticCategories_storageAccount — PASS (163s)
```

---

## Related issues / PRs

| Link | Description |
|---|---|
| Fixes [#16195](https://github.com/hashicorp/terraform-provider-azurerm/issues/16195) | Customer request: V2 API connection support |
| [hashicorp/pandora#5543](https://github.com/hashicorp/pandora/pull/5543) | Pandora workaround — injects `kind` into `ApiConnectionDefinition` |
| [Azure/azure-rest-api-specs#45544](https://github.com/Azure/azure-rest-api-specs/issues/45544) | Upstream spec fix (open) |
| [Azure/bicep#3512](https://github.com/Azure/bicep/issues/3512) | Same root cause in Bicep (open since 2021) |

---

## Checklist

- [x] `go build ./...` — clean
- [x] `go vet ./internal/services/connections/...` — clean
- [x] `go vet ./internal/services/monitor/...` — clean
- [x] `gofmt -l` — no files flagged
- [x] Acceptance tests run and passing
- [x] Documentation updated (resource + data source)
- [x] `ForceNew: true` set — matches Azure API immutability constraint
- [x] Backward compatible — existing configs without `kind` default to `"V1"`, no plan diff
