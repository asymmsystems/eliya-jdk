# Changelog

All notable releases of Eliya are documented here. Newest first.

## 25.0.3-eliya - 2026-06-10 (first GA)

**Upstream baseline:** OpenJDK 25.0.3-ga (released 2026-04-21). See the [upstream release notes](https://wiki.openjdk.org/display/JDKUpdates/JDK+25+Updates).

**Security and CPU backports beyond upstream:** None. Eliya 25.0.3 ships the upstream 25.0.3-ga security set unchanged. The next quarterly CPU (25.0.4) follows the upstream cadence.

**Eliya additions:**
- `-XX:EliyaProfile=Production` flag (`ccstr` enum, ADR-00001). When set, activates six Phase 1 operational-readiness ergonomics: heap-dump-on-OOM with structured path, exit-on-OOM, Native Memory Tracking summary, predictable crash log path (ADR-00006), container support reinforced, and diagnostic VM options unlocked.
- `-XX:EliyaConflictCheck` flag (`bool`, default `true`, ADR-00001).
- `asymm` CLI installed at `bin/asymm`.
- `/var/log/eliya/` directory created by RPM and DEB post-install.

**Deferred to Phase 2:** Continuous JFR and unified GC logging are Phase 2 work, not active in 25.0.3.

**Provenance:**
- `conf/security/java.security` is bit-identical to upstream (PATCHES.md §6).
- Release artefacts signed with the Eliya release key (Ed25519, ADR-00002).
- Builds are bit-for-bit reproducible from the source tarball attached to this release.
