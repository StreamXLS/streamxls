# Install + first run

The authoritative, step-by-step install and first-run guide — with SmartScreen and signature-verification screenshots — is at **[streamxls.com/download](https://streamxls.com/download)** and **[streamxls.com/docs](https://streamxls.com/docs)**. This page is a short summary.

**Requirements:** Windows 10 / 11 · Microsoft Excel for Windows (32- or 64-bit) · .NET Framework 4.8 (present by default on modern Windows) · TWS or IB Gateway with API access enabled · TWS API **10.47.01** or newer.

## Install

1. Download **`StreamXLS-Setup-<version>.exe`** from the [Releases](https://github.com/StreamXLS/streamxls/releases) page. It is signed by **StreamXLS LLC**; before running it you can right-click → *Properties* → **Digital Signatures** and confirm the signer reads *StreamXLS LLC*. Each installer's SHA-256 is published with the release (the asset's digest) and at [streamxls.com/download](https://streamxls.com/download) for independent verification (`Get-FileHash -Algorithm SHA256 <path>`).
2. Windows SmartScreen may warn on first run (the certificate is still building reputation at current download volume). [streamxls.com/download](https://streamxls.com/download) shows exactly what the expected prompt looks like and how to proceed.
3. Run the installer. It installs **per-user** to `%LOCALAPPDATA%\StreamXLS` (no admin rights), registers the COM server with ProgID `Tws.Rtd` under `HKCU\Software\Classes`, and places the demo workbook. The same installer works for both 32-bit and 64-bit Excel. It is removable from **Settings → Apps**.
4. StreamXLS binds to *your own* entitled TWS API install at runtime — the TWS API DLL (`CSharpAPI.dll`) is **not** shipped. The supported TWS API floor is **10.47.01** (see [streamxls.com/docs-config](https://streamxls.com/docs-config)).

## First run

- Open the demo workbook from **Start menu → *StreamXLS demo workbook***. This is the recommended copy: the installer places it locally, so it carries no Mark-of-the-Web and Excel runs its signed macros without prompting. (A copy downloaded from the web is blocked by Excel until you right-click the file → Properties → **Unblock**.)
- In TWS or IB Gateway, enable API access: **Configure → API → Settings → Enable ActiveX and Socket Clients**, and add `127.0.0.1` to trusted IPs for a single-host setup.
- Once TWS is logged in and the socket port matches, the first `=RTD(...)` formula resolves to a live value. The **StreamXLS Control Panel** (Start menu) shows license status, TWS API status, and settings — it is the first place to look when something isn't behaving.

Default socket ports: TWS `7496` live / `7497` paper; IB Gateway `4001` live / `4002` paper.

## Workbooks built against the IBKR sample

The IBKR Excel RTD sample registers as `Tws.TwsRtdServerCtrl`; StreamXLS registers its own ProgID, `Tws.Rtd`, and the two coexist on the same machine with no conflict. To migrate an existing workbook, replace the sample's ProgID in its formulas with `Tws.Rtd`.
