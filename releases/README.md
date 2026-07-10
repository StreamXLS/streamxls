# Releases

Signed binary releases of StreamXLS publish to the [GitHub Releases](https://github.com/StreamXLS/streamxls/releases) page of this repository (not to this directory). Verification steps — publisher identity and checksum — are documented at [streamxls.com/download](https://streamxls.com/download).

**v1.0.0 is live.** Each release attaches the per-user installer `StreamXLS-Setup-<version>.exe`, signed by **StreamXLS LLC**, with its SHA-256 published in the release notes.

## Verifying a download

1. **Signature.** Right-click the `.exe` → *Properties* → **Digital Signatures**; the signer should read *StreamXLS LLC* and the certificate should report as valid. Windows SmartScreen may still warn on first run while the certificate builds reputation at current volume — this is expected; [streamxls.com/download](https://streamxls.com/download) shows the exact prompt.
2. **Checksum.** Compare the SHA-256 in the release notes against your local hash:
   ```powershell
   Get-FileHash -Algorithm SHA256 path\to\StreamXLS-Setup-X.Y.Z.exe
   ```

Release notes call out any changes since the previous release, including any breaking changes to the topic-string schema.
