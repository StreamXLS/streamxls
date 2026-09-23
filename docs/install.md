# Install + first run

The authoritative, step-by-step install and first-run guide — with SmartScreen and signature-verification screenshots — is at **[streamxls.com/download](https://streamxls.com/download)** and **[streamxls.com/docs](https://streamxls.com/docs)**. This page is a summary.

**Requirements:**

- Windows 10 / 11
- Microsoft Excel for Windows (32- or 64-bit)
- Interactive Brokers [TWS](https://www.interactivebrokers.com/en/trading/download-tws.php) or [IB Gateway](https://www.interactivebrokers.com/en/trading/ibgateway-latest.php)
- **[TWS API](https://interactivebrokers.github.io/) v10.47.01** or newer.
- The Microsoft Visual C++ 2015–2022 Redistributable ([x64](https://aka.ms/vs/17/release/vc_redist.x64.exe); [x86](https://aka.ms/vs/17/release/vc_redist.x86.exe) as well if you run 32-bit Excel). Nearly every Windows PC already has this; a freshly built Windows Server image may not.

## Install

1. Download **`StreamXLS-Setup-<version>.exe`** from the [Releases](https://github.com/StreamXLS/streamxls/releases) page. It is signed by **StreamXLS LLC**; before running it you can right-click → *Properties* → **Digital Signatures** and confirm the signer reads *StreamXLS LLC*. Each installer's SHA-256 is published with the release (the asset's digest) and at [streamxls.com/download](https://streamxls.com/download) for independent verification (`Get-FileHash -Algorithm SHA256 <path>`).
2. Windows SmartScreen may warn on first run (the certificate is still building reputation at current download volume). [streamxls.com/download](https://streamxls.com/download) shows exactly what the expected prompt looks like and how to proceed.
3. Run the installer.

## First run

- Open your TWS or IB Gateway and enable API access: **File → Global Configuration → API → Settings → Enable ActiveX and Socket Clients**.
- Open the demo workbook from the Start menu → **StreamXLS Control Panel**.  Pick any sheet and click `Activate Formulas (Live)` to start the streaming data.

---

## Administrator installs

By default StreamXLS installs for the user who launches the installer.  This requires no admin rights, puts the software in `%LOCALAPPDATA%\StreamXLS`, and registers the COM server under `HKCU\Software\Classes`.

**On a Windows machine you use through the built-in `Administrator` account: choose "Install for all users."**  This puts it in `Program Files` and registers it under `HKLM\Software\Classes`. Windows does not let a program running with full administrator rights use an installation registered to one user, and on that account *every* program runs that way — including Excel — so a single-user install simply fails to load StreamXLS. Installing from a **named administrator account** you created yourself also works. This is the single most common cause of a completely dead sheet on a trading VPS; see [Every StreamXLS cell shows `#N/A`](FAQ.md#every-streamxls-cell-shows-na--on-a-vps-or-when-excel-runs-as-administrator) for the full explanation. The same applies on any machine if you launch Excel with "Run as administrator" — start it normally, or install for all users.

## Pre-installing StreamXLS in a Windows image

Ensure that the image has the **Microsoft Visual C++ 2015–2022 Redistributable (x64)** ([download](https://aka.ms/vs/17/release/vc_redist.x64.exe)). A bare Windows Server image does not.

Install StreamXLS using:

```text
StreamXLS-Setup-<version>.exe /ALLUSERS /VERYSILENT /NORESTART
```

and nothing else — do not start Excel, do not open the Control Panel, and do not accept the licence agreement on the template.

Each person who later signs in to a machine built from the image accepts the licence agreement themselves, the first time they open the StreamXLS Control Panel (**Start menu → StreamXLS**). Until they do, StreamXLS formulas show a licence message instead of data —
`Accept the StreamXLS licence agreement in the StreamXLS Control Panel (Start menu → StreamXLS)`
— and `=RTD("Tws.Rtd",, "LICENSE_STATE")` reads `AssentRequired`.

The Control Panel also checks for a newer version at that moment and offers it, so an image pinned to one release still starts each customer on current bits. The update is always offered, never installed silently.
