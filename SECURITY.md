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

## Version-string derivation

The JEP 322 BUILD field carries different meanings across Eliya's lifecycle.

All 25.0.x builds prior to 25.0.4 derive the BUILD field from the Eliya source-mirror's commit count since upstream GA. The currently-published `eliya-jdk 25.0.3` ships as `25.0.3+24`, where `+24` is the count of mirror commits since `jdk-25.0.3-ga`.

From 25.0.4 onward, the JEP 322 BUILD field carries upstream's GA promoted-build coordinate, and Eliya's per-build identity is carried in the JEP 322 `$OPT` field as `rN`. The 25.0.4 first GA will ship with `java.runtime.version` = `25.0.4+7-r1`, where `+7` is upstream's GA promoted-build and `r1` is Eliya's first promoted build of that source base. The 25.0.4 first respin would ship as `25.0.4+7-r2`. The banner reads `OpenJDK Runtime Environment Eliya-25.0.4 (build 25.0.4+7-r1)`.

Under both schemes, the canonical per-build pin is the SHA-256 of the published artefact, signed in `SHA256SUMS.txt.asc`, or the OCI image digest for container pulls. The version-string derivation does not affect this pin: the SHA-256 of the bytes is identical regardless of how the version string was composed.

The currently-published 25.0.3 GitHub Release is immutable. Any subsequent respin (e.g. an out-of-cycle CVE response before the 25.0.4 cutover) will be published as a new Release tag, not as an in-place update to the existing 25.0.3 Release.

## Further Reading

- [Security overview](https://asymm.systems/product/eliya/security.html) - the published Eliya security story (forensic observability, supply-chain provenance, Phase 4 compliance profiles)
- [Verify-download walkthrough](https://asymm.systems/product/eliya/user-guide/verify-download.html) - step-by-step operator guide with worked examples
- [PATCHES.md](PATCHES.md)
