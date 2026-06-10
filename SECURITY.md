# Security Policy

## Supported Versions

| Version | Supported | Patched through |
|---|---|---|
| Eliya 25.x LTS | Yes | September 2029 |

Eliya publishes LTS lines only - currently JDK 25, with JDK 29 LTS to be added at its GA (September 2027 expected). Each line is supported for the duration of the upstream OpenJDK Vulnerability Group's 4-year commitment (Eliya 25 through September 2029). Non-LTS feature releases (JDK 26, 27, 28, …) are not published - see [Why Eliya ships JDK 25 only](https://asymm.systems/research/choosing-a-jdk-2026.html#why-eliya-ships-jdk-25-only) for the trade-off rationale.

## Reporting a Vulnerability

If you discover a security vulnerability in Eliya, please report it responsibly:

**Email:** security@asymm.systems

Please include:

- Description of the vulnerability
- Steps to reproduce
- Which Eliya version(s) are affected
- Whether this is an Eliya-specific issue or an upstream OpenJDK issue

## Patch Commitment

Eliya follows OpenJDK's quarterly security patch cadence (January, April, July, October). When OpenJDK releases a security update:

1. We rebuild Eliya against the patched upstream within 1–2 weeks.
2. Signed release artefacts (`tar.gz`, `.deb`, `.rpm`, `SHA256SUMS.txt`, `SHA256SUMS.txt.asc`) are published to GitHub Releases.
3. Multi-arch container images are published to GHCR (`ghcr.io/asymmsystems/eliya-jdk:25-lts`, `:25.0.N`).
4. The SDKman candidate list is updated (`sdk install java <new-version>-eliya`).
5. `CHANGELOG.md` records the release and its upstream baseline.

## Upstream Issues

If a vulnerability exists in upstream OpenJDK (not Eliya-specific), please also report it to the [OpenJDK Vulnerability Group](https://openjdk.org/groups/vulnerability/) directly.

## Release Signatures

Every Eliya release artefact (`tar.gz`, `.deb`, `.rpm`, `SHA256SUMS.txt`) is signed with the Eliya release key.

- **Signing key UID:** `Eliya Releases (Asymm Systems) <eliya@asymm.systems>`
- **Public key:** [`https://key.asymm.systems/eliya-signing-key.asc`](https://key.asymm.systems/eliya-signing-key.asc)
- **Fingerprint:** *(published with first Eliya release)*

### Verification

Verification is **two-step**: cross-check the public key's fingerprint against an independent source before trusting it, then use the trusted key to verify the signed checksums.

**Step 1 - Fetch the public key and cross-check its fingerprint.**

```bash
curl -fsSL https://key.asymm.systems/eliya-signing-key.asc | gpg --import
gpg --fingerprint eliya@asymm.systems
```

The reported fingerprint **must** match the canonical value published independently on:

- [verify-download page](https://asymm.systems/product/eliya/user-guide/verify-download.html) on the Eliya website
- [security page](https://asymm.systems/product/eliya/security.html) on the Eliya website
- `keys.openpgp.org` - `gpg --keyserver keys.openpgp.org --recv-keys <fingerprint>` returns the same key
- `keyserver.ubuntu.com` - `gpg --keyserver keyserver.ubuntu.com --recv-keys <fingerprint>` returns the same key

If any of these channels disagrees with the fingerprint reported by `gpg --fingerprint`, **stop and report** to `security@asymm.systems`. A mismatch means either the key file at `key.asymm.systems` has been substituted, your network path has been tampered with, or this `SECURITY.md` itself is forged.

**Step 2 - Verify the signed checksums and the artefact.**

```bash
gpg --verify SHA256SUMS.txt.asc SHA256SUMS.txt
sha256sum -c SHA256SUMS.txt
```

A successful `gpg --verify` (key matches Step 1's cross-checked fingerprint, signature valid) plus a clean `sha256sum -c` is the only attestation Eliya makes about an artefact's authenticity. Trust no Eliya download that does not verify against both steps.

## Further Reading

- [Security overview](https://asymm.systems/product/eliya/security.html) - the published Eliya security story (forensic observability, supply-chain provenance, Phase 4 compliance profiles)
- [Verify-download walkthrough](https://asymm.systems/product/eliya/user-guide/verify-download.html) - step-by-step operator guide with worked examples
- [PATCHES.md](PATCHES.md)
