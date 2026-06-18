# Releases

Signed binary releases of StreamXLS will publish to the [GitHub Releases](https://github.com/StreamXLS/streamxls/releases) page of this repository (not to this directory).

This directory holds the documentation that accompanies releases — changelog notes, upgrade guides, hash-verification instructions for unsigned builds during the pre-cert window.

## Current status

Signed releases are gated on code-signing certificate onboarding. Until that completes, this directory is a placeholder.

## What to expect

When the first release lands:

1. The `Releases` tab of this repository will show a new tag (e.g. `v1.0.0`).
2. Attached artifact: a signed `.msi` installer. The publisher identity on the signature will be `StreamXLS LLC`.
3. SHA-256 hash of the `.msi` will be published alongside the release notes for independent verification.
4. Release notes will reference any relevant changes since the previous release and call out any breaking changes to the topic-string schema.

## Pre-cert workaround (interim)

If an unsigned binary publishes before the Trusted Signing cert is in place:

- The `.msi` will be flagged by Windows SmartScreen as an unsigned installer.
- The release notes will publish the SHA-256 hash. Verify locally:
  ```powershell
  Get-FileHash -Algorithm SHA256 path\to\downloaded\StreamXLS-X.Y.Z.msi
  ```
  and confirm the value matches what is published on the release page.
- This window is interim — the goal is signed-by-default.
