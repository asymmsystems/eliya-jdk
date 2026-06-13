# Eliya Patches

This document describes the patch surface Eliya adds on top of upstream OpenJDK 25 source. The upstream source passes the Java SE 25 Technology Compatibility Kit (TCK), but TCK conformance is a property of a specific built binary, not of source code; the Eliya binary's own TCK run under the OpenJDK Community TCK License Agreement (OCTLA) is a Phase 2 deliverable. Patches are maintained on the `eliya/25` branch of a private mirror; a public diff URL will be added here once that mirror is public-readable. Until then, security researchers and packagers can request the patch series at `security@asymm.systems`.

## 1. Flag Taxonomy

Eliya introduces two flags in Phase 1:

| Flag | Type | Default | Valid Values | Purpose |
|---|---|---|---|---|
| `EliyaProfile` | `ccstr` | `"None"` | `"None"`, `"Production"` (Phase 1); Phase 4 values reserved | Layer 2 profile selector (enum syntax) |
| `EliyaConflictCheck` | `bool` | `true` | `true`/`false` | Toggle for the three-tier conflict detector |

`EliyaProfile` is a Hotspot `ccstr` rather than a boolean precisely because the Hotspot parser then enforces profile mutual exclusivity structurally (last-wins semantics). Profile values beyond `None` and `Production` are reserved for future phases and produce a startup error if set in Phase 1.

## 2. Diagnostic Path Resolution

Eliya computes diagnostic-output paths at JVM startup from environment and command-line input. Each component (base path, service name, replica name) follows a strict precedence chain - first non-empty value wins.

### Base path

1. `ELIYA_DIAGNOSTIC_PATH` environment variable
2. `-Deliya.diagnostic.path=` system property
3. Platform default: Linux `/var/log/eliya`, macOS `/usr/local/var/eliya`, FreeBSD `/var/log/eliya`

### Service name

1. `ELIYA_SERVICE_NAME` environment variable
2. `-Deliya.service.name=` system property
3. `HOSTNAME` environment variable (K8s sets this to the pod name)
4. Literal fallback: `"default"`

### Replica name

1. `ELIYA_REPLICA_NAME` environment variable
2. `-Deliya.replica.name=` system property
3. `HOSTNAME` environment variable
4. Unset - replica directory level is suppressed

## 3. Path Layout

Path components compose into either a three-level or two-level layout (canonical reference: ADR-00006):

- **Three-level** (replica meaningfully distinct from service):
  `${base}/${service}/${replica}/${category}/`
- **Two-level** (replica equals service, or replica is unset):
  `${base}/${service}/${category}/`

**Replica suppression rule.** The replica directory is suppressed when the resolved replica is unset, or when the resolved replica string is byte-equal to the resolved service string. This handles two collapse cases cleanly:

- Bare-metal single-JVM where nothing is configured and both service and replica fall through to the same `HOSTNAME` - collapses to `${base}/${HOSTNAME}/${category}/`.
- Fully-configured K8s where `ELIYA_SERVICE_NAME=billing` and `HOSTNAME=billing-pod-7xk29p` differ - keeps both levels: `/var/log/eliya/billing/billing-pod-7xk29p/jfr/`.

Crash dumps follow the same layout under a `crash/` category, producing a file path of the form `${...}/crash/hs_err_pid%p.log` for `ErrorFile`.

## 4. Profile Activation

When `EliyaProfile=Production`, Eliya sets the following defaults - *only* when the user has not set the flag explicitly on the command line. This preserves user-explicit-wins semantics: any value the operator passes on the command line beats Eliya's default.

### Phase 1 (shipped 25.0.3, six ergonomics activated via `FLAG_SET_ERGO`)

1. **Heap dump on OOM** - `HeapDumpOnOutOfMemoryError=true`, with `HeapDumpPath` set to `build_eliya_path("heap-dumps")`.
2. **Exit on OOM** - `ExitOnOutOfMemoryError=true`. The process exits non-zero when an `OutOfMemoryError` is thrown, letting orchestrators (systemd, Kubernetes) restart on a clean baseline rather than degrade in place.
3. **Native Memory Tracking summary** - `NativeMemoryTracking="summary"`.
4. **Predictable crash log path** - `ErrorFile` set to `build_eliya_error_file_path()`; `CreateCoredumpOnCrash=true`. The path-bearing flags at items 1 and 4 use the `service/replica/category` layout from sections 2-3 (collapsing to `service/category` when replica attribution is not meaningful), so post-mortem artefacts land in a predictable location without per-deployment configuration.
5. **Container support reinforced** - `UseContainerSupport=true`. No-op when already set; recorded explicitly so operators auditing flags see the value originated from `EliyaProfile=Production`.
6. **Unlocked diagnostic VM options** - `UnlockDiagnosticVMOptions=true`, enabling subsequent diagnostic flags operators may want.

### Phase 2 additions (planned)

Two additional defaults are planned for Phase 2 alongside the bundled diagnostic-tools work. These cannot use `FLAG_SET_ERGO` (the activation mechanism is not a product flag); they need `JfrOptionSet` and `LogConfiguration` hooks.

- **Continuous JFR** - `FlightRecorder=true` plus a `StartFlightRecording` recording spec (`disk=true, maxage=24h, maxsize=250m, dumponexit=true`) using `build_eliya_path("jfr")`.
- **Unified GC logging** - unified-logging GC configuration writing under `build_eliya_path("gc")` with rotation.

## 5. Three-Tier Conflict Detection

Conflict detection runs at JVM startup. Gated by `-XX:-EliyaConflictCheck` for advanced operators experimenting with flag combinations or running CI/CD flag matrices.

- **Tier 1 - Silent resolution.** User-explicit command-line settings override profile-implied defaults (existing Hotspot semantics; no message).
- **Tier 2 - Warning.** Redundant flag combinations (profile already activates a capability that the user also explicitly enables) emit a stderr warning prefixed `[Eliya] Warning:`. JVM continues normally.
- **Tier 3 - Fatal.** Profile + capability negation that breaks the profile's invariants emits `[Eliya] Fatal:` and aborts startup. The message names the conflict, the profile invariant, and the three escape hatches (`EliyaProfile=None`, drop the negation, compose capabilities directly).

**Phase 1 scope:** the matrix is empty. Phase 1 ships only `EliyaProfile=Production`, `EliyaProfile=None`, and the six profile-set ergonomics; all six respect `FLAG_IS_CMDLINE` and resolve as tier-1 silent overrides. No tier-2 or tier-3 events can fire because there are no opposing capability flags yet.

`-XX:EliyaConflictCheck` ships in 25.0.3 to establish the public flag namespace and the dispatch integration point before they carry weight. Operators baking JVM flags into deploy scripts today can adopt `-XX:-EliyaConflictCheck` now and not have to revise the command line when Phase 2 capability flags land. The dispatch is gated by the flag and calls `check_flag_consistency()` whose body populates as Phase 2 capability flags and Phase 4 profile values introduce real conflict cases. Profile-vs-profile collisions are impossible by syntax (the `ccstr` flag has last-wins semantics).

## 6. Minimal Patch Surface

Eliya's patch surface is intentionally surgical. Minimal upstream divergence is a design principle, not a side effect: by avoiding deep modifications to the HotSpot VM, Eliya delivers no Java SE API divergence (the two new flags `EliyaProfile` + `EliyaConflictCheck` are `-XX` JVM options, not Java SE APIs), minimises fork-drift risk, and keeps each quarterly upstream CPU mergeable within days rather than weeks.

Eliya source files live in a separate mirror tree alongside the upstream sources; upstream-file intrusion is limited to single-line delegations that are touched once and never again across phases. This keeps each quarterly upstream CPU a mechanical merge rather than a per-release reconciliation, and keeps the auditable diff between upstream OpenJDK 25 and Eliya 25 small by construction.

**Security posture predictability** - `conf/security/java.security` is bit-identical to upstream. Cryptographic policy, TLS algorithm enablement, JCE configuration, certificate trust anchors, and the disabled-algorithms lists are precisely as upstream OpenJDK 25 ships them. Operators auditing crypto/TLS policy can rely on upstream's documented behaviour without re-baselining against an Eliya-specific defaults file.

Eliya's Phase 1 security value lies in forensic observability defaults activated by `EliyaProfile=Production` (the six ergonomics enumerated in section 4) and supply-chain provenance (signed releases per ADR-00002). Continuous JFR and unified GC logging are Phase 2 work, not active in 25.0.3.

**Security tightening** - when it happens - is the explicit job of Phase 4 compliance profiles (`EliyaProfile=PCIDSS`, `=HIPAA`, `=SOX`, `=FedRAMP`, `=GDPR`, `=ISO27001`, `=SOC2`, plus three combined profiles for cross-framework coverage). These profiles are: (a) opt-in by design - none activate without an explicit operator-set flag; (b) framework-aligned - each profile's tightening maps to the named compliance framework's control requirements; (c) layered on top of upstream defaults, never by editing `java.security` in place. The two-layer flag taxonomy in ADR-00001 exists precisely to make this opt-in tightening expressible and auditable. The baseline file stays predictable; tightening becomes a deliberate operator choice, not a hidden Eliya default.

## 7. Runtime requirements (portability floor)

Eliya runs on every mainstream Linux distribution from 2011 onward. Per ADR-00023 rev. 2026-06-02 the floors are **per-architecture**, set by upstream OpenJDK `make/devkit/Tools.gmk`'s OL-branch defaults (verbatim, no Eliya override):

**Linux x86_64:** `glibc >= 2.12` (Oracle Linux 6.4 sysroot)
- Covers RHEL / CentOS / Oracle Linux 6+, Ubuntu 12.04+, Debian 7+, Amazon Linux 1+, SUSE / SLES 12+, and derivatives.
- Wider compatibility surface than Temurin's 2.17 floor on x86_64.

**Linux aarch64:** `glibc >= 2.17` (Oracle Linux 7.6 sysroot)
- Covers RHEL / CentOS / Oracle Linux 7+, Ubuntu 16.04+, Debian 8+, Amazon Linux 2+, SUSE / SLES 12+, and derivatives.
- arm64 production Linux floors at 2.17 (no glibc 2.12 on arm64).

Verify on the target: `ldd --version` reports the installed glibc.

**Other invariants (same on both arches):**

- **No separate `libstdc++` requirement** - the C++ runtime is linked statically (`--with-stdc++lib=static`), so there is no GLIBCXX floor; only glibc governs portability.
- Minimum Linux **kernel 2.6.32** on both arches (both OL6.4 and OL7.6 ship glibc built with `--enable-kernel=2.6.32`).

The per-arch floor is pinned at build time by an OpenJDK GCC 14 devkit carrying the matching per-arch OL sysroot, and **enforced on every shipped binary** by `07c` (`objdump`: max `GLIBC_` symbol `<=` the per-arch ceiling and zero `GLIBCXX_`; `readelf` `.note.ABI-tag` kernel `<=` 2.6.32). The runtime-`dlopen`'d native libraries (fontconfig, X11, ALSA, CUPS) are declared as `.deb`/`.rpm` dependencies so the package manager pulls them on the target. See ADR-00023 for the full rationale and native-dependency matrix.
