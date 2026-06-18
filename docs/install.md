# Install + first run

> **Status:** placeholder. Verified install and first-run instructions will publish with the first signed Releases drop, once code-signing certificate onboarding completes. The steps below are illustrative only and are **not** a tested install procedure.

## What the install will look like (preview)

1. Download the signed installer `.msi` from the [Releases](https://github.com/StreamXLS/streamxls/releases) page of this repository. Verify the signature is by `StreamXLS LLC`.
2. Run the installer. It copies `StreamXLS.dll` and its dependencies to the install path and registers the COM component with ProgID `Tws.Rtd`.
3. (Optional, see below) Create the legacy-ProgID alias `Tws.TwsRtdServerCtrl` if any existing workbooks reference the IBKR-sample ProgID.
4. Configure TWS or IB Gateway with API access enabled (Configure → API → Settings → Enable ActiveX and Socket Clients; add `127.0.0.1` to trusted IPs).
5. Open the example workbook under [`examples/`](../examples/) in Excel. The first `=RTD(...)` formula should resolve to a live quote once TWS is logged in and the socket port matches the workbook configuration.

## Optional legacy-ProgID alias

The IBKR sample registers as `Tws.TwsRtdServerCtrl`. This server uses the more concise `Tws.Rtd` as its default ProgID, but ships an alias script that adds a registry entry pointing the legacy `Tws.TwsRtdServerCtrl` ProgID at this server's CLSID — letting existing workbooks built against the IBKR sample keep working without modification.

To create the alias after the main install, run `CreateAlias.ps1` from the installed `tools\` directory:

```powershell
# Per-user alias (regular PowerShell window):
.\tools\CreateAlias.ps1

# Or system-wide alias (administrator PowerShell window):
# (same command, elevated)
```

The alias is a registry entry only; it does not duplicate the DLL. If the underlying CLSID is later unregistered (`regasm /unregister` or uninstall), the alias entry remains in the registry but no longer resolves until the CLSID is registered again.

## Pre-cert workaround

Before the Trusted Signing cert lands, an unsigned-binary distribution may publish with SHA-256 hashes posted on the Releases page. SmartScreen will flag the unsigned installer; verifying the hash against the value published on this repository's Releases page is the manual integrity check during that window.

## What you need on the IBKR side

- Live IBKR account or a paper-trading account (`Edit → Global Configuration → API → Settings → Master API client ID`).
- TWS (latest stable) or IB Gateway installed and logged in.
- API socket port consistent between TWS and the workbook (TWS default `7496` live / `7497` paper; IB Gateway default `4001` live / `4002` paper).
- Trusted-IP entry for the host that runs Excel (typically `127.0.0.1` for single-host setups).

Full IBKR-side prerequisites are documented in the [TWS API documentation](https://www.interactivebrokers.com/campus/ibkr-api-page/twsapi-doc/); they apply to StreamXLS in the same way they apply to any TWS API client.
