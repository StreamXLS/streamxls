# Install + first run

The authoritative, step-by-step install and first-run guide — with SmartScreen and signature-verification screenshots — is at **[streamxls.com/download](https://streamxls.com/download)** and **[streamxls.com/docs](https://streamxls.com/docs)**. This page is a summary.

**Requirements:** Windows 10 / 11 · Microsoft Excel for Windows (32- or 64-bit) · .NET Framework 4.8 (present by default on modern Windows) · TWS or IB Gateway with API access enabled · **[TWS API](https://interactivebrokers.github.io/) v10.47.01** or newer.

## Install

1. Download **`StreamXLS-Setup-<version>.exe`** from the [Releases](https://github.com/StreamXLS/streamxls/releases) page. It is signed by **StreamXLS LLC**; before running it you can right-click → *Properties* → **Digital Signatures** and confirm the signer reads *StreamXLS LLC*. Each installer's SHA-256 is published with the release (the asset's digest) and at [streamxls.com/download](https://streamxls.com/download) for independent verification (`Get-FileHash -Algorithm SHA256 <path>`).
2. Windows SmartScreen may warn on first run (the certificate is still building reputation at current download volume). [streamxls.com/download](https://streamxls.com/download) shows exactly what the expected prompt looks like and how to proceed.
3. Run the installer. It installs **per-user** to `%LOCALAPPDATA%\StreamXLS` (no admin rights), registers the COM server with ProgID `Tws.Rtd` under `HKCU\Software\Classes`, and places the demo workbook. The same installer works for both 32-bit and 64-bit Excel. It is removable from **Settings → Apps**.

## First run

- Open your TWS or IB Gateway and enable API access: **File → Global Configuration → API → Settings → Enable ActiveX and Socket Clients**.
- Open the demo workbook from the Start menu → **StreamXLS Control Panel**.  Pick any sheet and click `Activate Formulas (Live)` to start the streaming data.
