# Root Cause Analysis

## What is `ui5appruntime`?

`ui5appruntime.html` is a **server-generated runtime page dedicated to running a single app**, produced by the SaaS approuter (`cp.portal`) of SAP Build Work Zone (BWZ) / Cloud Portal.

In BWZ, each app is launched embedded as an **iframe** inside the Shell, and the content of that iframe is this `ui5appruntime.html`. Its main responsibilities are:

- **Booting a lightweight ushell (AppRuntime)** — as indicated by `data-sap-ui-oninit="module:sap/ushell/appRuntime/ui5/AppRuntime"` in the response HTML, it boots a "minimal ushell for running just one app" rather than the full FLP. The loaded `sap/fiori/appruntime-min-0.js` through `-3.js` are its implementation.
- **Resolving the UI5 bootstrap version** — it receives the `sap-ui-version` from the request and resolves it to a concrete patch version on the server side to assemble the bootstrap `src`. In a successful response, `sap-ui-version=1.147` is resolved to `https://sapui5.hana.ondemand.com/1.147.2/resources/...`, i.e. `1.147` becomes `1.147.2`.
- **Injecting Shell-to-app integration settings** — it embeds settings such as Personalization, Container, and `appIndex` (`sap.cf.modules.tcprovider`) into the HTML so the in-iframe app can communicate with the parent Shell.

In short, it is a "**server-generated iframe host page for launching exactly one app on a specified UI5 version**". Unless the version to use is determined, the page itself cannot be generated.

## Why the Error Occurs: Duplicate `sap-ui-version`

The decisive difference between the success and error cases is the **number of `sap-ui-version` parameters**.

Success case (noversion) — `sap-ui-version` appears only once:
```
...&sap-ui-app-id=noversion&...&sap-ui-version=1.147
```

Error case (version136) — `sap-ui-version` appears **twice**, with conflicting values:
```
...&sap-ui-version=1.136.x&sap-ui-version=1.147&sap-ui-app-id=version136&...
                 ^^^^^^^^                ^^^^^
              from manifest           from BWZ URL
```

| Source | Value | Origin |
|---|---|---|
| App manifest | `1.136.x` | `sap.platform.cf.ui5VersionNumber` in `version136/webapp/manifest.json` |
| BWZ URL parameter | `1.147` | `sap-ui-version=1.147&sap-iframe-params=sap-ui-version` in the site URL |

The `noversion` manifest has no `sap.platform.cf` block, while only `version136`/`version145` have `ui5VersionNumber` — that is the dividing line.

The flow that occurs is:

1. When an app has `sap.platform.cf.ui5VersionNumber: "1.136.x"`, the approuter **appends the app-declared version `sap-ui-version=1.136.x`** to the `ui5appruntime.html` URL.
2. On the BWZ side, because of `sap-iframe-params=sap-ui-version`, the Shell URL's `sap-ui-version=1.147` is **also propagated and appended to the iframe**.
3. As a result, two parameters with the same name appear, and the server-side version resolution logic receives contradictory input (`1.136.x` and `1.147`). On top of that, `1.136.x` is a wildcard notation rather than a concrete patch, so it cannot be reconciled with a fixed value like `1.147`.
4. Since `ui5appruntime.html` cannot generate the bootstrap URL when the version cannot be resolved, **an exception is thrown during HTML generation, resulting in a 500 Internal Server Error**.

`noversion` works because there is only one version source (BWZ's `1.147`), so there is no conflict and the server resolves it cleanly to `1.147.2`.

## Conclusion / Workaround

**Specifying the version in two places — the "URL parameter" and the "manifest (`sap.platform.cf.ui5VersionNumber`)" — causes a version conflict when `ui5appruntime` is generated, resulting in a 500.** Use only one of them.

- **To pin for the whole site** → specify it via the URL parameter (`sap-ui-version`) or in the site settings, and remove `sap.platform.cf.ui5VersionNumber` from the manifest.
- **To pin per app** → put a **concrete patch (e.g. `1.136.19`)** in the manifest's `ui5VersionNumber` instead of a wildcard like `1.136.x`, and do not add `sap-ui-version` on the BWZ URL side.

The direct trigger is `sap-iframe-params=sap-ui-version` in the site URL. Because it makes the manifest-derived version conflict, do not combine this URL parameter when you operate by pinning the version on the app side.

References:
- [SAP KBA 3353289 - Work Zone "Internal Server Error" when navigating to HTML5 applications](https://userapps.support.sap.com/sap/support/knowledge/en/3353289)
- [SAP KBA 2916153 - Errors when opening apps on Launchpad, Work Zone or Cloud Portal service](https://userapps.support.sap.com/sap/support/knowledge/en/2916153)
