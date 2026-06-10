# Changelog

All notable releases of Eliya are documented here. Newest first.

## 25.0.3-eliya - 2026-05-14 (first public release)

- Based on TCK-certified upstream OpenJDK 25.0.3-ga (released
  2026-04-21).
- Introduces the Eliya flag taxonomy (per ADR-00001):
  `-XX:EliyaProfile=Production` (ccstr enum, Phase 1) and
  `-XX:EliyaConflictCheck` (bool, default `true`).
- Activates eight observability defaults when
  `EliyaProfile=Production`: continuous JFR (Phase 2), heap-dump-on-OOM,
  NMT summary, GC logs (Phase 2 continuous-by-default), container awareness, crash dump
  generation, adaptive diagnostic path layout (per ADR-00006),
  unlocked diagnostic VM options.
- `java.security` is bit-identical to upstream - see PATCHES.md §6.
- Release artefacts signed with the Eliya release key (per ADR-00002).
