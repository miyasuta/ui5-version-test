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
- Combining methods 1 and 2 caused an error (even when the URL parameter and the app specified the same version) — see [docs/root-cause-analysis.md](docs/root-cause-analysis.md)

## Recommendations
- **To pin for the whole site** → use the URL parameter (`sap-ui-version`) or the site settings only, and remove `sap.platform.cf.ui5VersionNumber` from the manifest.
- **To pin per app** → put a concrete patch (e.g. `1.136.19`) in the manifest's `ui5VersionNumber`, and do not add `sap-ui-version` on the BWZ URL.
- **Do not combine** the two — it causes a 500 error (see Root Cause Analysis).
- On a **"Spaces and Pages - New Experience"** site, do not set the shell to a version below 1.146 via the URL parameter, or the site itself fails to load (the Group layout is unaffected).

## Test Apps
Three apps differ only in how they declare their UI5 version:

| App | `minUI5Version` | `sap.platform.cf.ui5VersionNumber` |
|---|---|---|
| noversion | 1.148.1 | (none) |
| version136 | 1.136.0 | 1.136.x |
| version145 | 1.145.0 | 1.145.x |

Only `version136` / `version145` have a `sap.platform.cf` block — this is the dividing line that triggers the conflict described in the Root Cause Analysis.

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

The shell resolves to 1.147.2. `noversion` follows the shell (1.147.2), but `version136` / `version145` error out because the manifest-declared version conflicts with the URL parameter.

## Why Dual Specification Fails

When an app pins its version in the manifest **and** the version is also supplied via the BWZ URL parameter, the request for `ui5appruntime` ends up carrying **two `sap-ui-version` parameters** — one from the manifest-declared version and one propagated from the site URL. The server cannot resolve a single version from these conflicting duplicates, producing a 500 Internal Server Error.

![Internal Server Error](docs/images/internal%20server%20error.png)

See [docs/root-cause-analysis.md](docs/root-cause-analysis.md) for the full root cause and workaround.
