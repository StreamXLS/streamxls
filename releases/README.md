# Releases

Signed binary releases of StreamXLS publish to the [GitHub Releases](https://github.com/StreamXLS/streamxls/releases) page of this repository (not to this directory), created by the `streamxls` app on behalf of StreamXLS LLC. Each release attaches the per-user installer `StreamXLS-Setup-<version>.exe`, signed by **StreamXLS LLC**.

## Verifying a download

1. **Signature.** Right-click the `.exe` → *Properties* → **Digital Signatures**; the signer should read *StreamXLS LLC* and the certificate should report as valid. Windows SmartScreen may still warn on first run while the certificate builds reputation at current volume — this is expected; [streamxls.com/download](https://streamxls.com/download) shows the exact prompt.
2. **Checksum.** The signature above is the authenticity check — it proves the bytes came from StreamXLS LLC. The SHA-256 adds an integrity cross-check from a second origin: [streamxls.com/download](https://streamxls.com/download) publishes each release's hash independently of GitHub, and GitHub shows each asset's own digest beside the download. Compare against your local hash:
   ```powershell
   Get-FileHash -Algorithm SHA256 path\to\StreamXLS-Setup-X.Y.Z.exe
   ```

Release notes call out changes since the previous release, including any change to the topic-string schema. Installed copies are also offered each release through the product's update channel — updates are notified, never forced, and cryptographically verified before they run.
