# Eliya Patches

This document describes the patch surface Eliya adds on top of TCK-certified upstream OpenJDK. Patches are maintained on the `eliya/25` branch of a private mirror; a public diff URL will be added here once that mirror is public-readable. Until then, security researchers and packagers can request the patch series at `security@asymm.systems`.

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

1. **Continuous JFR** - `FlightRecorder=true` plus a `StartFlightRecording` recording spec (`disk=true, maxage=24h, maxsize=250m, dumponexit=true`) using `build_eliya_path("jfr")`.
2. **Heap dump on OOM** - `HeapDumpOnOutOfMemoryError=true`, with `HeapDumpPath` set to `build_eliya_path("heap-dumps")`.
3. **Native Memory Tracking summary** - `NativeMemoryTracking="summary"`.
4. **Always-on GC logs** - unified-logging GC configuration writing under `build_eliya_path("gc")` with rotation.
5. **Container awareness** - `UseContainerSupport=true` (reinforces upstream default).
6. **Crash dump generation** - `ErrorFile` set to `build_eliya_error_file_path()`; `CreateCoredumpOnCrash=true`.
7. **Adaptive diagnostic path layout** - all path-bearing flags above use the layout from §2–§3, giving ADR-00006's `service/replica/category` structure (collapsing to `service/category` when replica attribution is not meaningful) without per-deployment configuration.
8. **Unlocked diagnostic VM options** - `UnlockDiagnosticVMOptions=true`, enabling subsequent diagnostic flags operators may want.

## 5. Three-Tier Conflict Detection

Conflict detection runs at JVM startup. Gated by `-XX:-EliyaConflictCheck` for advanced operators experimenting with flag combinations or running CI/CD flag matrices.

- **Tier 1 - Silent resolution.** User-explicit command-line settings override profile-implied defaults (existing Hotspot semantics; no message).
- **Tier 2 - Warning.** Redundant flag combinations (profile already activates a capability that the user also explicitly enables) emit a stderr warning prefixed `[Eliya] Warning:`. JVM continues normally.
- **Tier 3 - Fatal.** Profile + capability negation that breaks the profile's invariants emits `[Eliya] Fatal:` and aborts startup. The message names the conflict, the profile invariant, and the three escape hatches (`EliyaProfile=None`, drop the negation, compose capabilities directly).

The matrix is small in Phase 1 (only `Production` and `None` exist) and grows as Phase 2 capability flags and Phase 4 profile values land. Profile-vs-profile collisions are impossible by syntax - the `ccstr` flag has last-wins semantics.

## 6. Minimal Patch Surface

Eliya's patch surface is intentionally surgical. Minimal upstream divergence is a design principle, not a side effect: by avoiding deep modifications to the HotSpot VM, Eliya guarantees zero API divergence, eliminates the risk of fork drift, and keeps each quarterly upstream CPU mergeable within days rather than weeks.

Per ADR-00009 (source file layout), Eliya source files live in a **separate top-level mirror tree** at `src/eliya/hotspot/share/...` parallel to upstream's `src/hotspot/share/...`. Tests live at `test/eliya/hotspot/jtreg/...` per ADR-00011. Upstream-file intrusion is reduced to **single-line delegations** that are touched once and never again across phases.

### Eliya-owned source files (separate tree, all new)

| Eliya file | Role |
|---|---|
| `src/eliya/hotspot/share/runtime/eliya.{hpp,cpp}` | Top-level facade - `Eliya::apply()` dispatched from upstream `apply_ergo()` |
| `src/eliya/hotspot/share/runtime/eliyaArguments.{hpp,cpp}` | Diagnostic-path resolution, production-profile activator, three-tier conflict checker |
| `src/eliya/hotspot/share/runtime/flags/eliyaFlags.{hpp,cpp}` | Data-driven `EliyaProfile` validation + activation table (`KNOWN_PROFILES[]`); `EliyaProfileConstraintFunc` body |
| `test/eliya/hotspot/jtreg/runtime/Eliya/EliyaProfileValidation.java` | JTREG: constraint accepts None/Production; rejects all 10 Phase 4 reserved names |
| `test/eliya/hotspot/jtreg/runtime/Eliya/EliyaDiagnosticPathTest.java` | JTREG: env-over-sysprop precedence, replica suppression, three-level vs two-level paths |

### Upstream-file intrusion (touched once, then never again)

| Upstream file | Change | Lines |
|---|---|---|
| `src/hotspot/share/runtime/globals.hpp` | Two flag declarations (`EliyaProfile` ccstr, `EliyaConflictCheck` bool) + `constraint(EliyaProfileConstraintFunc, AtParse)` directive | 12 |
| `src/hotspot/share/runtime/flags/jvmFlagConstraintsRuntime.hpp` | One entry added to the `RUNTIME_CONSTRAINTS` macro list: `f(ccstr, EliyaProfileConstraintFunc)` | 1 |
| `src/hotspot/share/runtime/arguments.cpp` | `#include "runtime/eliya.hpp"` + single-line `Eliya::apply();` call at end of `apply_ergo()` | 2 |
| `make/hotspot/lib/JvmFlags.gmk` | `JVM_SRC_ROOTS += $(TOPDIR)/src/eliya/hotspot` so the build glob picks up the Eliya mirror tree (per ADR-00009 §2.2) | 1 |

**Approximately 16 lines** of upstream-file modification total, all of which are touched once during ISSUE-00001 and never again across phases. Adding a Phase 4 profile activator changes only `eliyaFlags.cpp`'s `KNOWN_PROFILES[]` table + `eliyaArguments.cpp` activator body. Adding a new diagnostic category (Phase 1.5+) changes only `eliyaArguments.cpp`'s `Category` enum + `CATEGORY_NAMES[]` array.

**Security posture predictability** - `conf/security/java.security` is bit-identical to upstream. Cryptographic policy, TLS algorithm enablement, JCE configuration, certificate trust anchors, and the disabled-algorithms lists are precisely as upstream OpenJDK 25 ships them. Operators auditing crypto/TLS policy can rely on upstream's documented behaviour without re-baselining against an Eliya-specific defaults file.

Eliya's Phase 1 security value lies in forensic observability defaults (continuous JFR, heap-dump-on-OOM, crash-dump generation, GC logs - see §4) and supply-chain provenance (signed releases per ADR-00002).

**Security tightening** - when it happens - is the explicit job of Phase 4 compliance profiles (`EliyaProfile=PCIDSS`, `=HIPAA`, `=SOX`, `=FedRAMP`, `=GDPR`, `=ISO27001`, `=SOC2`, plus three combined profiles for cross-framework coverage). These profiles are: (a) opt-in by design - none activate without an explicit operator-set flag; (b) framework-aligned - each profile's tightening maps to the named compliance framework's control requirements; (c) layered on top of upstream defaults, never by editing `java.security` in place. The two-layer flag taxonomy in ADR-00001 exists precisely to make this opt-in tightening expressible and auditable. The baseline file stays predictable; tightening becomes a deliberate operator choice, not a hidden Eliya default.

## 7. Runtime requirements (portability floor)

Eliya's Linux binaries are built against an old-glibc sysroot (ADR-00023) so a single artefact runs across the supported distribution range:

- **glibc >= 2.17** (x86_64 and aarch64) - covers RHEL / CentOS / Oracle Linux 7+, Ubuntu 16.04+, Debian 8+, Amazon Linux 2, SUSE / SLES 12+, and derivatives. (`ldd --version` reports the installed glibc.)
- **No separate `libstdc++` requirement** - the C++ runtime is linked statically (`--with-stdc++lib=static`), so there is no GLIBCXX floor; only glibc governs portability.
- Minimum Linux **kernel 2.6.32** (the glibc-2.17 / OL7 pairing).

The floor is pinned at build time by an OpenJDK GCC 14 devkit carrying a glibc-2.17 sysroot, and **enforced on every shipped binary** by `07c` (`objdump`: max `GLIBC_` symbol <= 2.17 and zero `GLIBCXX_`; `readelf` `.note.ABI-tag` kernel <= 2.6.32) - validated end-to-end on x86_64 (`libjli.so` GLIBC_2.14, `bin/asymm` GLIBC_2.4). The runtime-`dlopen`'d native libraries (fontconfig, X11, ALSA, CUPS) are declared as `.deb`/`.rpm` dependencies so the package manager pulls them on the target. See ADR-00023 for the full rationale and native-dependency matrix.
