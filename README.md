# Verifying the UI5 Version

## Purpose
Investigate how to pin the UI5 version of a UI5 app running in SAP Build Work Zone (BWZ).

> Tested: 2026-06, on a `us10` trial Work Zone site. UI5 versions served by the platform (e.g. the latest patch) move over time, so the concrete numbers below are point-in-time measurements.

## Methods
1. Pin via `sap.platform.cf.ui5VersionNumber` in manifest.json
2. When opening SAP Build Work Zone, specify the URL parameter `&sap-ui-version=<version>&sap-iframe-params=sap-ui-version`

## Summary
- With either method 1 or 2, the app ran on the specified UI5 version
- With method 1, a different UI5 version could be specified per app
- Combining methods 1 and 2 works **only if both sides specify the exact same concrete patch** (e.g. `1.136.18` in the manifest and `sap-ui-version=1.136.18` in the URL). Any mismatch — including a wildcard like `1.136.x` vs a fixed value — produces a 500 error. See [docs/root-cause-analysis.md](docs/root-cause-analysis.md)

Choose based on **who owns the version**:

- **Pin per app (recommended) — set `sap.platform.cf.ui5VersionNumber` in the manifest.** A wildcard `1.136.x` (auto-follows the latest patch of the line) or a concrete patch `1.136.18` (fully frozen) both work when used alone. This pins the app iframe, not the shell, so it is safe under both the Group and Spaces & Pages layouts. *Honored only by the Work Zone / CF launchpad runtime, not by a standalone `index.html`.*
- **Control centrally — set the shell version via `sap-ui-version` (URL parameter or site settings) and leave `ui5VersionNumber` out of the manifests.** Lets you bump all apps at once. On a **"Spaces and Pages - New Experience"** site, keep the shell at **1.146 and above**, or the site itself fails to load (the Group layout is unaffected).
- **Avoid combining the two.** It works only when both sides carry the byte-identical concrete patch — i.e. the same patch hard-coded in two places and kept in sync. Any drift (or a wildcard) gives a 500. Pick a single source instead.

## Test Apps
Three apps differ only in how they declare their UI5 version:

| App | `minUI5Version` | `sap.platform.cf.ui5VersionNumber` |
|---|---|---|
| noversion | 1.148.1 | (none) |
| version136 | 1.136.0 | 1.136.x |
| version145 | 1.145.0 | 1.145.x |

Only `version136` / `version145` have a `sap.platform.cf` block — this is the dividing line that triggers the conflict described in the Root Cause Analysis.

> The table and the main results below reflect the baseline config (`1.136.x`). For the dual-specification test ([Combining manifest pin + URL parameter](#combining-manifest-pin--url-parameter)), version136's `ui5VersionNumber` was later changed to the concrete patch `1.136.18` — the current value in the repo.

<details>
<summary>manifest.json snippets</summary>

- noversion
```
  "sap.ui5": {
    ...
    "dependencies": {
      "minUI5Version": "1.148.1",
      ...
    },
```

- version136
```
  "sap.ui5": {
    ...
    "dependencies": {
      "minUI5Version": "1.136.0",
      ...
    },
  "sap.platform.cf": {
        "ui5VersionNumber": "1.136.x"
  }    
```

- version145
```
  "sap.ui5": {
    ...
    "dependencies": {
      "minUI5Version": "1.145.0",
      ...
    },
  "sap.platform.cf": {
        "ui5VersionNumber": "1.145.x"
  }  
```

</details>

## Verification Results

Resolved UI5 version per app and runtime:

| App | Local | HTML5 Applications | BWZ (default shell 1.148.1) | BWZ + `sap-ui-version=1.147` |
|---|---|---|---|---|
| noversion | 1.148.1 | 1.149.0 | 1.148.1 | 1.147.2 |
| version136 | 1.136.0 | 1.149.0 | 1.136.19 | ❌ error |
| version145 | 1.145.0 | 1.149.0 | 1.145.3 | ❌ error |

### Running Locally
The default UI5 version is the one specified in `minUI5Version` in manifest.json.

### Running in HTML5 Applications (index.html)
All three apps load the **same, latest** version (1.149.0). This is because the app is launched through its own `index.html`, which bootstraps UI5 **without specifying a version**: the `sap-ui-version` URL parameter is not present, and `sap.platform.cf.ui5VersionNumber` is honored only by the Work Zone / CF launchpad runtime — not by the standalone `index.html`. With no version pinned, the HTML5 Applications runtime serves the latest available version.

### Running in BWZ
The shell runs on its default version (1.148.1), and each app resolves to the version declared in its manifest.

### Specifying the Version via BWZ URL Parameter
Specified `&sap-ui-version=1.147&sap-iframe-params=sap-ui-version`.
Example: `https://3d7fa4ddtrial.launchpad.cfapps.us10.hana.ondemand.com/site?siteId=57d3a691-61c5-4675-9459-3fb6ef65eab6&sap-ui-version=1.147&sap-iframe-params=sap-ui-version#Shell-home`

The shell resolves to 1.147.2. `noversion` follows the shell (1.147.2), but `version136` / `version145` error out because the manifest-declared version cannot be reconciled with the URL parameter (see below).

### Combining manifest pin + URL parameter
What actually decides success is whether the two `sap-ui-version` values can collapse to a single version:

| Manifest `ui5VersionNumber` | URL `sap-ui-version` | Result |
|---|---|---|
| `1.136.x` (wildcard) | `1.147` | ❌ 500 |
| `1.145.x` (wildcard) | `1.145` | ❌ 500 |
| `1.136.18` (concrete) | `1.136.18` | ✅ opens on 1.136.18 |

Combining works **only when both values are the byte-identical concrete patch**. A wildcard (`1.136.x`) or any difference between the two values cannot be resolved to one version → 500. Note `1.145.x` vs `1.145` fails even though both resolve to the same patch (1.145.3): the literal strings differ, so the reconciliation never happens. Matching the exact patch on both sides is possible but fragile, so it is not a recommended strategy.

## Why Dual Specification Fails

When an app pins its version in the manifest **and** a version is also supplied via the BWZ URL parameter, the request for `ui5appruntime` carries **two `sap-ui-version` parameters** — one from the manifest and one propagated from the site URL. The server can still serve the page **if the two values are the identical concrete patch** (they collapse to a single version). But if they differ in any way — a wildcard like `1.136.x` vs a fixed value, or two different versions — it cannot resolve a single version and returns a **500 Internal Server Error**.

![Internal Server Error](docs/images/internal%20server%20error.png)

See [docs/root-cause-analysis.md](docs/root-cause-analysis.md) for the full root cause and workaround.
