# SONiC EOL & Deprecation Plan — 202611

## Table of Content

- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions/Abbreviations](#3-definitionsabbreviations)
- [4. Overview](#4-overview)
- [5. Why remove code](#5-why-remove-code)
- [6. Process](#6-process)
- [7. Candidates for 202611](#7-candidates-for-202611)
  - [7.1 Remove now](#71-remove-now)
    - [Barefoot / Tofino platform (with DTEL)](#barefoot--tofino-platform-with-dtel)
    - [Standalone P4 platform](#standalone-p4-platform)
    - [Nephos platform](#nephos-platform)
    - [GoBGP FPM container](#gobgp-fpm-container)
    - [Python 2 packages](#python-2-packages)
    - [Kubernetes master](#kubernetes-master)
    - [docker-basic_router](#docker-basic_router)
    - [System Telemetry container](#system-telemetry-container)
    - [Broken FRR config modes](#broken-frr-config-modes)
    - [Old Debian build leftovers](#old-debian-build-leftovers)
  - [7.2 Deprecate now, remove later](#72-deprecate-now-remove-later)
    - [Bullseye base containers — remove in 202705](#bullseye-base-containers--remove-in-202705)
    - [FRR split-unified config mode — remove in 202705](#frr-split-unified-config-mode--remove-in-202705)
    - [REST API — remove in 202711](#rest-api--remove-in-202711)
- [8. Config and management impact](#8-config-and-management-impact)
- [9. Warmboot/Fastboot impact](#9-warmbootfastboot-impact)
- [10. Testing](#10-testing)
- [11. Additional findings](#11-additional-findings)

### 1. Revision

| Rev | Date | Author | Change |
|-----|------|--------|--------|
| 0.1 | 2026-07-19 | Brad House (Nexthop) | First draft. Security WG. |

### 2. Scope

This HLD does two things:

1. Sets up a repeatable, per-release process to deprecate and remove SONiC features.
2. Lists the features we propose to act on in **202611**.

It comes out of the SONiC Security Working Group. Goal: shrink the attack surface. Less code means fewer packages, fewer CVEs, less to build and test.

Releases ship twice a year: **November (`YYYY11`)** and **May (`YYYY05`)**. So the cycles here are 202611 (Nov 2026), 202705 (May 2027), 202711 (Nov 2027).

### 3. Definitions/Abbreviations

| Term | Meaning |
|------|---------|
| Deprecate | Mark a feature as going away. Still ships, but warns of removal. |
| Remove | Delete the code and its build wiring. |
| SBOM | Software Bill of Materials. |
| DTEL | Data-plane telemetry (in-band / INT). |
| TAM | Telemetry and Monitoring (added in 202605). |
| EOL | End of life. |
| CI | The community build/test gate. |

### 4. Overview

SONiC still ships features that few or no one runs. Some are dead. Some broke when a dependency moved on. Some were tied to hardware that no longer exists. Each one still adds packages to the image, findings to every CVE scan, and jobs to CI.

This HLD proposes a standing process to retire such features, plus the first batch to act on.

### 5. Why remove code

- The best fix for a CVE is to delete the code that carries it.
- Unmaintained code is a standing risk. It rarely gets patched.
- Dead build wiring hides real problems and slows builds.
- Fewer features = smaller image, fewer scans to triage, less CI.

### 6. Process

Per release:

1. **Propose.** File (or update) this HLD with candidates. Each candidate says what it is, what replaces it, the gap (what you lose), and a verdict.
2. **Two verdicts:**
   - **Remove now** — dead, broken, or EOL with no real users. Delete this cycle.
   - **Deprecate now, remove later** — still has possible users or needs migration work. Name a removal release.
3. **Review.** The community reviews it. This HLD gets scheduled at the weekly HLD review (Tuesdays). Component owners sign off. A feature is not removed without owner sign-off.
4. **Announce.** Deprecations go in the release notes.
5. **Remove** in the named release.

Rules of thumb:

- "Not built by default" and "not in CI" are **hints**, not proof. We check real use before we call anything dead.
- A forced bump (libyang, base image, submodule pointer, mass format) does **not** count as maintenance.
- If a replacement has a gap, we name it so users can speak up.

### 7. Candidates for 202611

Each candidate below stands on its own: what it is, why it goes, and anything to watch out for.

#### 7.1 Remove now

##### Barefoot / Tofino platform (with DTEL)

`platform/barefoot`. Intel ended the Tofino line, and the platform is absent from the CI build matrix (aspeed_arm64, broadcom, marvell-prestera, mellanox, nvidia_bluefield, vs, alpinevs, vpp) — so it is unbuilt and untested. It's about 7.7 MB / 880 files and pulls in the Barefoot SAI, saithrift, `docker-syncd-bfn`, `docker-saiserver-bfn`, and the ODM sub-platforms built on Tofino (Accton Wedge100BF, Arista 7170, Ingrasys, Netberg, WNC).

Data-plane telemetry (DTEL, `dtelorch`) only ran on Tofino in practice, so it goes with the platform. TAM, added in 202605, covers the same need. Also delete the stale `barefoot` job still sitting in `azure-pipelines-build.yml`.

##### Standalone P4 platform

`platform/p4` — the old bmv2 software behavioral model (`docker-sonic-p4`, `sai-p4-bm`, `p4c-bm`, `tenjin`, `p4-hlir`). It's a reference target, not real hardware. Dead since around 2022, in no CI, about 12 MB / 920 files with 4 stale external submodules. The only thing referencing it outside `platform/p4/` is a Debian-8-era TODO in `rules/redis.mk`.

Remove `platform/p4/*`, its submodules, and that TODO. Leave `rules/p4lang.mk` alone — the p4lang toolchain (bmv2/p4c/PI) it builds is a live dependency of DASH SAI, which ships in `sonic-vs.img.gz`, and of the PTF test containers.

##### Nephos platform

`platform/nephos`. The Nephos/MediaTek silicon vendor left the market around 2019. About 2.3 MB / 160 files (`docker-syncd-nephos`, SAI, saithrift, ODM modules). There's no `device/nephos` tree, no genuine commit in years, and it isn't in CI. Cleanup is mostly its `azure-pipelines-build.yml` entry.

##### GoBGP FPM container

`docker-fpm-gobgp`, `src/gobgp`, `rules/gobgp.*`. SONiC standardized on FRR (`SONIC_ROUTING_STACK = frr`). The gobgp image-build rules were deleted back in 2021, so it hasn't been buildable as an image since — yet the build still compiles a 2017-vintage Go GOBGP `.deb` that nothing installs.

Remove the container, `src/gobgp`, the `rules/gobgp.*` recipes, and the dead `DOCKER_FPM_GOBGP` and `DOCKER_FPM_QUAGGA` references in `rules/docker-fpm.*`. Nothing is lost — FRR does all of this.

##### Python 2 packages

`swsssdk-py2` and `redis-dump-load-py2`. Both are gated behind `ENABLE_PY2_MODULES`, which is off for bullseye, bookworm, and trixie — so neither is built on any current release. Pure dead weight. (The py3 version of swsssdk is a different case — it's still in use, so it's out of scope here and noted under Additional findings.)

##### Kubernetes master

`INCLUDE_KUBERNETES_MASTER`. This runs a full Kubernetes control plane on the switch — apiserver, controller, scheduler, proxy, etcd 3.5.0, coredns 1.8.4, and the dashboard 2.7.0 web UI — plus cloud credential libraries for backups. Every one of those is years past end of life, so this is the single biggest CVE-surface win in the list. It's off by default and has no real CI.

This is master only. The worker feature (`INCLUDE_KUBERNETES`) is separate and stays — someone hardened its cluster-join security before this WG existed, which points to a real user. Removing master also drops `sonic-kubernetes_master.yang`. Leave the `ctrmgrd` container wrapper alone; it runs for ordinary feature start/stop, not just k8s.

##### docker-basic_router

A SAI demo/reference container. It's wired into no build rule, never ships in an image, and isn't in CI. Its last real change was 2020. Nothing uses it — straight delete.

##### System Telemetry container

`docker-sonic-telemetry`, off by default. It's redundant: this container and the gnmi container run the exact same binary, `/usr/sbin/telemetry` (started by `telemetry.sh` and `gnmi-native.sh` respectively). There is no separate gNMI server — the `telemetry` binary *is* the server; the name is just historical. Both read the same `TELEMETRY|gnmi` config, and the gnmi start script only adds ZMQ and VRF options on top. So there is no functional gap: Get/Set/Subscribe and dial-out are identical.

Remove the container and the `INCLUDE_SYSTEM_TELEMETRY` flag. Keep the `telemetry` binary — the gnmi container runs it.

##### Broken FRR config modes

`docker_routing_config_mode` has four values. `separated` and `split` write per-daemon config files (`bgpd.conf`, `zebra.conf`, and so on). FRR 10.5.4, which SONiC ships, dropped support for per-daemon config files — so both of these modes are broken today. This is really dead-code removal.

The supported path is `unified`, where config comes from bgpcfgd and `config_db.json`; make that the default. Right now `minigraph.py` hardcodes the default as `separated` (and init_cfg doesn't set it, so it resolves to `separated`), so this change must flip that default to `unified`. The fourth mode, `split-unified`, still works and is handled under Deprecate.

##### Old Debian build leftovers

Debian 8 (jessie), 9 (stretch), and 10 (buster) are all long past end of life. The risk is someone building a new container on a 6+-year-old base.

stretch and buster each have real `docker-base-*`, `docker-config-engine-*`, and `docker-swss-layer-*` containers to delete. jessie is different — there's no `docker-base-jessie`. What's left of jessie is the `sonic-slave-jessie` build slave and its `make jessie` target, plus jessie strings in the generic `docker-base` (armhf/arm64 sources and `FROM ...:jessie`); drop the slave and its wiring, and check the generic `docker-base` before touching it.

Before deleting any base, confirm nothing still bases a container on it. Bullseye (Debian 11) is newer and handled under Deprecate.

#### 7.2 Deprecate now, remove later

##### Bullseye base containers — remove in 202705

`docker-base-bullseye`, `docker-config-engine-bullseye`, `docker-swss-layer-bullseye` (Debian 11). Newer than the bases above, and may still be a live base for some containers, so give it a cycle: move those containers to bookworm/trixie, then remove the bullseye bases in 202705.

##### FRR split-unified config mode — remove in 202705

The manual routing mode, where the operator writes `frr.conf` by hand. It has a real problem: the bgp container runs supervisord, not systemd, so FRR can't hot-reload — any `frr.conf` change forces a full FRR restart, which drops BGP sessions. That makes it impractical for production, so it's doubtful anyone serious runs it.

Deprecate now and consolidate on `unified` (bgpcfgd + `config_db.json`). Before removing, confirm bgpcfgd and `config_db.json` cover the FRR features operators actually need.

##### REST API — remove in 202711

`docker-sonic-restapi`, off by default. Its own spec calls it the "SONiC REST API for Baremetal Scenarios" — an imperative agent for baremetal VNET/VXLAN/VLAN config over HTTPS, not a general REST interface. gNMI is the go-forward, but two things gNMI doesn't cover: bulk route programming with per-route partial success (HTTP 207), and route expiry (timed route aging).

Because of that gap, give it the longest runway. Before removing, confirm no control plane still drives it. The mgmt-framework REST server is a different thing and stays.

### 8. Config and management impact

Removing a feature drops its `FEATURE` table entries, build flags, and any YANG models tied to it (for example `sonic-kubernetes_master.yang`). Users who never enabled these see no CLI change.

The FRR change removes `separated` and `split` from `docker_routing_config_mode` and makes `unified` the default. Platform removals don't affect other platforms.

### 9. Warmboot/Fastboot impact

None. Every candidate is off by default, dead, or specific to hardware being removed. Nothing here touches the warmboot/fastboot path on supported platforms.

### 10. Testing

- Each supported CI platform still builds after the removals.
- Removed packages are gone from the SBOM and CVE scan (image diff).
- The default FRR path (`unified`) comes up and programs routes.
- All remaining containers build with nothing pointing at a deleted base.

### 11. Additional findings

Real, but out of scope here — removing them needs code porting, which this HLD doesn't cover. Noted so they're tracked.

- **swsssdk-py3.** The old Python Redis SDK. `swsscommon` replaces it, but code still imports `swsssdk` at runtime (`sonic-utilities/scripts/dualtor_neighbor_check.py`, `sonic-py-common`, the vpp container). It can't be deprecated until those are ported to `swsscommon`.
