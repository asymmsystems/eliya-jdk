# Eliya

**The forensic-grade JVM platform from Asymm Systems for compliance-conscious production.**

Eliya is an OpenJDK distribution from Asymm Systems engineered for compliance-conscious production in regulated industries — operations that demand continuous observability, precise diagnostic attribution, and audit-ready artifacts. **Four phases of differentiation:** operational-readiness defaults today; continuous observability, bundled local diagnostic tooling, cross-artefact correlation analysis, and compliance-aligned profiles ahead.

Canonical home: [asymm.systems/product/eliya](https://asymm.systems/product/eliya).

---

## What Eliya is

The JVM platform foundation for Asymm Systems' compliance-tooling ecosystem, structured as a phased roadmap rather than a single-point product:

| Phase | Status | What ships |
|---|---|---|
| **Phase 1** | NOW | Production observability defaults activated by a single flag — Java Flight Recorder always-on, heap dump on OOM, native memory tracking, GC logging, predictable diagnostic paths under `/var/log/eliya/${SERVICE}/`. All running locally; no SaaS dependencies. Linux x86_64 + aarch64 + FreeBSD Research Preview. `asymm` CLI minimum surface. |
| **Phase 2** | 2026-2027 | Bundled local diagnostic tools (Eclipse MAT headless, async-profiler), FIPS variant for BFSI/healthcare/government procurement, hosted apt/yum repositories, macOS aarch64 binary, `asymm eliya` profiling and heap analysis subcommands. |
| **Phase 3** | Post-Series-A | [Asymm Forensics](https://asymm.systems/product/forensics) — JVM diagnostic forensics platform with cross-artefact correlation analysis, ML-based pattern detection, compliance reporting (PCI DSS, HIPAA, SOX), Dial-aware analysis. |
| **Phase 4** | Post-Dial OaC expansion | Compliance-aligned profiles — defaults aligned with the framework your industry requires. |

Built from the same upstream OpenJDK source tree. Quarterly CPU cadence preserved — every upstream security patch flows into Eliya within two weeks. API semantics inherited from upstream; an application that runs on upstream OpenJDK runs on Eliya. The HotSpot VM underneath is upstream; Eliya contributes the strategic-direction overlay, the operational packaging, and the audit-ready release pipeline.

[Dial](https://asymm.systems/product/dial) — the TMF/ODA-native programming language from Asymm Systems — targets any JDK 25+: Corretto, Temurin, Zulu, Oracle, Liberica, Eliya. Eliya is the recommended runtime because its observability defaults align with how Dial workflows are typically diagnosed; many users adopt the pairing for that reason.

## Available Versions

| Version | Base | LTS Window | Status |
|---|---|---|---|
| **Eliya 25** | OpenJDK 25.0.3 | through September 2029 | Latest |

Eliya ships a single LTS line. JDK 25 is the current target because it is the newest LTS. JDK 21 is well-served by existing vendors; the marginal value of another JDK 21 build is low. JDK 29 LTS will be added at its GA (September 2027) with a 24-month overlap. **Non-LTS releases (26, 27, 28, 30, ...) are categorically never published** — the six-month upstream patch window is incompatible with Eliya's compliance-conscious positioning. See [ADR-00005](https://github.com/asymmsystems/asymm-jdk/blob/main/architecture/decision/adr-00005-jdk-version-targeting.md).

## Install

**SDKman**

```bash
sdk install java 25.0.3-eliya
```

**tar.gz** (manual install)

```bash
curl -fsSLO https://github.com/asymmsystems/eliya-jdk/releases/download/eliya-jdk-25.0.3/eliya-jdk-25.0.3-linux-x64.tar.gz
tar xzf eliya-jdk-25.0.3-linux-x64.tar.gz
export JAVA_HOME=$(pwd)/eliya-jdk-25.0.3
export PATH=$JAVA_HOME/bin:$PATH
java -version
```

**Debian / Ubuntu**

```bash
apt-get install ./eliya-jdk_25.0.3_amd64.deb
```

**RPM**

```bash
rpm -i eliya-jdk-25.0.3.x86_64.rpm
```

**Docker** (multi-arch, two tags)

```bash
# Version-pin (immutable per release):
docker pull ghcr.io/asymmsystems/eliya-jdk:25.0.3

# LTS-track (auto-moves with each quarterly refresh):
docker pull ghcr.io/asymmsystems/eliya-jdk:25-lts
```

Full install guides: [Linux](https://asymm.systems/product/eliya/user-guide/install-linux.html) · [Docker](https://asymm.systems/product/eliya/user-guide/install-docker.html) · [macOS](https://asymm.systems/product/eliya/user-guide/install-macos.html) · [FreeBSD](https://asymm.systems/product/eliya/user-guide/install-freebsd.html).

## Verify the release

Every Eliya artefact is signed against the canonical fingerprint:

```
076D E547 397A 5D27 EECE  E0B3 07A9 0689 B71A 158F
```

The signing key never resides on a networked host. Signatures are produced on an air-gapped operator host via a sneakernet flow.

```bash
# Import the public key:
gpg --keyserver keys.openpgp.org --recv-keys 076DE547397A5D27EECEE0B307A90689B71A158F

# Verify the signed checksums file:
gpg --verify SHA256SUMS.txt.asc SHA256SUMS.txt
# Expected: 'Good signature from "Eliya Releases (Asymm Systems) <eliya@asymm.systems>"'

# Verify each downloaded artefact:
sha256sum -c SHA256SUMS.txt --ignore-missing
```

Full verification walkthrough: [Verify your download](https://asymm.systems/product/eliya/user-guide/verify-download.html). Key-management details: [SECURITY.md](SECURITY.md).

## Usage

```bash
java -XX:EliyaProfile=Production -jar myapp.jar
```

The single flag activates Eliya's production-readiness defaults — continuous JFR (Phase 2), heap-dump-on-OOM, NMT summary, GC logs (Phase 2 continuous-by-default), container awareness, crash dump generation, three-level diagnostic path layout, unlocked diagnostic VM options. When something fails at 03:00, the last 24 hours of execution profile is already on disk.

JFR with the `default` profile carries **<1% CPU overhead** in typical production workloads — validated independently by Oracle, Datadog, New Relic, Microsoft, and Red Hat across the OpenJDK ecosystem since JFR went open-source in 2018. Full performance analysis: [Flags reference §Performance impact](https://asymm.systems/product/eliya/user-guide/flags-reference.html#performance-impact).

Flag breakdown + override semantics: [Flags reference](https://asymm.systems/product/eliya/user-guide/flags-reference.html). Two-layer flag architecture (Layer 1 capabilities + Layer 2 profile): [Flag architecture](https://asymm.systems/product/eliya/user-guide/flag-architecture.html).

## What's actually different from upstream OpenJDK

Engineers want a precise diff, not marketing tone.

**What's different:**
- One JVM flag: `-XX:EliyaProfile=Production` (off by default)
- Two helper functions in `arguments.cpp` for service name resolution and diagnostic-path construction
- The `asymm` CLI installed at `bin/asymm` — a native ELF launcher per [ADR-00020](https://github.com/asymmsystems/asymm-jdk/blob/main/architecture/decision/adr-00020-elf-launchers-and-execstack.md), executing the unified Asymm Systems CLI surface for diagnostics and operations
- `/var/log/eliya/` directory created by RPM/DEB postinstall

**What's inherited unchanged from upstream:**
- GC selection (JDK 25 ergonomics)
- JIT compiler tier transitions
- Class loading
- Module system
- Security provider order (SunJCE remains default)
- TLS protocol defaults
- All algorithm-disabling lists in `java.security`

Full enumeration: [Differences from upstream OpenJDK 25](https://asymm.systems/product/eliya/user-guide/diff-from-upstream.html). Patch manifest for auditors and downstream packagers (files, flags, upstream baselines): [PATCHES.md](PATCHES.md).

## Maintenance contract

Eliya refreshes at every upstream OpenJDK GA — once per quarter, within two weeks of upstream:

| Eliya release | Upstream base | Ship target |
|---|---|---|
| 25.0.3 (this) | OpenJDK 25.0.3 | First GA — June 2026 |
| 25.0.4 | OpenJDK 25.0.4 | ~2026-07-22 |
| 25.0.5 | OpenJDK 25.0.5 | ~2026-10-21 |
| 25.0.6 ... | quarterly per upstream | through JDK 25 LTS window (September 2029) |

When JDK 29 LTS arrives (September 2027 expected), Eliya 29 ships from its GA and runs in parallel with Eliya 25 through a 24-month overlap window. See [Lifecycle](https://asymm.systems/product/eliya/lifecycle.html) for the support schedule, [Refresh policy](https://github.com/asymmsystems/asymm-jdk/blob/main/research/fleeting/openjdk-25-update-cadence.md) for the operational cadence reference.

## Build provenance

Eliya is **built reproducibly from upstream source**. Customers can rebuild from [eliya-jdk-platform@release/25.0.3](https://github.com/asymmsystems/eliya-jdk-platform/tree/release/25.0.3) and verify byte-identical output via diffoscope.

Every release ships with `SHA256SUMS.txt.asc` covering all per-arch artefacts. The signing-key fingerprint above is the authoritative identifier of an Eliya release.

## Not sure Eliya is right for you?

[Choosing a JDK in 2026 — an honest guide](https://asymm.systems/research/choosing-a-jdk-2026.html) — vendor-by-vendor comparison across Temurin, Corretto, Zulu, Liberica, Oracle, Microsoft, Red Hat, SapMachine, IBM Semeru, GraalVM, and Eliya. Written by us, but honest about when Eliya is not the right pick.

## Documentation

- **Engineer** → [Flag architecture](https://asymm.systems/product/eliya/user-guide/flag-architecture.html) · [Flags reference](https://asymm.systems/product/eliya/user-guide/flags-reference.html) · [Differences from upstream](https://asymm.systems/product/eliya/user-guide/diff-from-upstream.html) · [Lessons from production](https://asymm.systems/product/eliya/user-guide/lessons-from-production.html)
- **Operator** → [Install guides](https://asymm.systems/product/eliya/user-guide/) · [Migration](https://asymm.systems/product/eliya/migration.html) · [Troubleshooting](https://asymm.systems/product/eliya/user-guide/troubleshooting.html) · [Integrations](https://asymm.systems/product/eliya/integrations.html) · [`asymm` CLI](https://asymm.systems/product/eliya/user-guide/cli.html)
- **Decider** → [Choosing a JDK in 2026](https://asymm.systems/research/choosing-a-jdk-2026.html) · [Security](https://asymm.systems/product/eliya/security.html) · [Lifecycle](https://asymm.systems/product/eliya/lifecycle.html) · [Roadmap](https://asymm.systems/product/eliya/roadmap.html) · [FAQ](https://asymm.systems/product/eliya/faq.html)
- **Auditor** → [PATCHES.md](PATCHES.md) · [SECURITY.md](SECURITY.md) · [CHANGELOG.md](CHANGELOG.md) · [Architecture Decision Records](https://github.com/asymmsystems/asymm-jdk/tree/main/architecture/decision)

## License

GNU General Public License, version 2, with the Classpath Exception (GPLv2+CE) — inherited from upstream OpenJDK. See [LICENSE](LICENSE). The Asymm overlay ([ADR-00020](https://github.com/asymmsystems/asymm-jdk/blob/main/architecture/decision/adr-00020-elf-launchers-and-execstack.md)) is licensed compatibly.

## Acknowledgments

Eliya is built on top of OpenJDK — a community effort by Oracle, Red Hat, IBM, Microsoft, Amazon, SAP, Tencent, BellSoft, Azul, the broader OpenJDK Adoption Group, and individual contributors. Eliya adds the strategic-direction overlay, operational packaging, signing discipline, the audit-ready provenance pipeline, and the Asymm CLI surface; the underlying Java platform is theirs. Thank you.

---

Copyright (c) 2026, Asymm Systems (Pvt) Ltd. Eliya is a product of Asymm Systems.
