# Eliya

**The forensic-grade JVM platform from Asymm Systems for compliance-conscious production.**

Eliya is an OpenJDK distribution from Asymm Systems engineered for compliance-conscious production in regulated industries - operations that demand continuous observability, precise diagnostic attribution, and audit-ready artifacts.

Canonical home: [asymm.systems/product/eliya](https://asymm.systems/product/eliya).

---

## Available Versions

| Version | Base | LTS Window | Status |
|---|---|---|---|
| **Eliya 25** | OpenJDK 25.0.3 LTS | through September 2029 | Latest LTS |

Eliya ships a single LTS line. JDK 25 is the current target because it is the newest LTS. JDK 21 is well-served by existing vendors; the marginal value of another JDK 21 build is low. JDK 29 LTS will be added at its GA (Sept 2027).

## Install

**SDKman**

```bash
sdk install java 25.0.3-eliya
```

**tar.gz** (manual install)

```bash
curl -fsSLO https://github.com/asymmsystems/eliya-jdk/releases/download/25.0.3-eliya/eliya-jdk-25.0.3-linux-x64.tar.gz
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

**Docker**

```bash
docker pull ghcr.io/asymmsystems/eliya:25-lts
```

Verify release artefacts against the Eliya signing key - see [SECURITY.md](SECURITY.md).

## Usage

```bash
java -XX:EliyaProfile=Production -jar myapp.jar
```

The single flag activates Eliya's production-readiness defaults (continuous JFR, heap-dump-on-OOM, NMT summary, always-on GC logs, container awareness, crash dump generation, three-level diagnostic path layout, unlocked diagnostic VM options). See [PATCHES.md](PATCHES.md) for the full list of what Eliya adds beyond upstream OpenJDK.

## Documentation

- [Eliya User Guide](https://asymm.systems/product/eliya/user-guide/) - the canonical documentation: install walkthroughs, configuration, profile reference, troubleshooting, lifecycle, migration
- [PATCHES.md](PATCHES.md) - exact patch manifest (files, flags, upstream baselines) for auditors and downstream packagers
- [SECURITY.md](SECURITY.md) - vulnerability disclosure address and release-signing key fingerprint
- [CHANGELOG.md](CHANGELOG.md) - machine-readable release history (keepachangelog format)
- `PLATFORMS.md` - support matrix (OS × architecture × tier) *(coming soon)*

## License

GNU General Public License, version 2, with the Classpath Exception (GPLv2+CE) - inherited from upstream OpenJDK. See [LICENSE](LICENSE).

---

Copyright (c) 2026, Asymm Systems (Pvt) Ltd. Eliya is a product of Asymm Systems.
