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
  - [7.2 Deprecate now, remove later](#72-deprecate-now-remove-later)
  - [7.3 Looked at, keeping](#73-looked-at-keeping)
- [8. Per-candidate detail](#8-per-candidate-detail)
- [9. Config and management impact](#9-config-and-management-impact)
- [10. Warmboot/Fastboot impact](#10-warmbootfastboot-impact)
- [11. Testing](#11-testing)
- [12. Open items](#12-open-items)

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
3. **Review.** Component owners and the TSC sign off. A feature is not removed without owner sign-off.
4. **Announce.** Deprecations go in the release notes.
5. **Remove** in the named release.

Rules of thumb:

- "Not built by default" and "not in CI" are **hints**, not proof. We check real use before we call anything dead.
- A forced bump (libyang, base image, submodule pointer, mass format) does **not** count as maintenance.
- If a replacement has a gap, we name it so users can speak up.

### 7. Candidates for 202611

#### 7.1 Remove now

| # | Candidate | Why | Watch out for |
|---|-----------|-----|---------------|
| A1 | Barefoot / Tofino platform (`platform/barefoot`) + DTEL | Intel EOL'd Tofino. Not in CI. Untested. | DTEL rides along (only Tofino ran it). Also drop the stale `barefoot` job in the pipeline YAML. |
| A2 | Standalone P4 platform (`platform/p4`) | Dead reference model since ~2022. In no CI. 12 MB, 923 files, 4 stale submodules. | Keep `rules/p4lang.mk` (bmv2/p4c/PI). DASH SAI and PTF still use it. |
| A3 | Nephos platform (`platform/nephos`) | Vendor gone (~2019). No real commits in years. No device tree. | Cleanup is mostly the pipeline YAML. |
| A4 | GoBGP FPM (`docker-fpm-gobgp`, `src/gobgp`, `rules/gobgp.*`) | Not buildable as an image since 2021, yet still compiles a 2017-era Go deb every build. FRR replaced it. | Drop the dead `DOCKER_FPM_GOBGP` / `DOCKER_FPM_QUAGGA` refs in `docker-fpm.*` too. |
| A5 | Python 2 packages (`swsssdk-py2`, `redis-dump-load-py2`) | Gated off on bullseye/bookworm/trixie. Never built on any current release. | None. Py3 versions stay (see B1). |
| A6 | Kubernetes master (`INCLUDE_KUBERNETES_MASTER`) | Runs a full k8s control plane on a switch. All pins are years EOL (k8s 1.22, etcd 3.5.0, coredns 1.8.4, dashboard 2.7.0) plus Azure creds. Off, no real CI. | Biggest CVE win. Master only — worker is separate (see B2). |
| A7 | `docker-basic_router` | Dead SAI demo container. Wired into no build. Never shipped. | Nothing uses it. |
| A8 | System Telemetry (`docker-sonic-telemetry`) | Off by default. The gnmi container reads the **same** config and does more. Already the default. | Confirm `gnmi_server` matches on Get/Set/Subscribe. Stop building the old `telemetry` binary too. |
| A9 | Broken FRR modes (`separated`, `split`) | Both write per-daemon FRR config files. FRR 10.5.4 (shipped) dropped support for those. They no longer work. | Which mode becomes the default is an open item — see [12](#12-open-items). |
| A10 | Old Debian build leftovers (jessie, stretch, buster) | jessie = Debian 8, stretch/buster = Debian 9/10. All long EOL. Risk: someone builds on a 6+ year old base. | jessie and stretch/buster are different shapes — see A10 note. Check nothing still `FROM`s a base before deleting. Bullseye stays for now. |

#### 7.2 Deprecate now, remove later

| # | Candidate | Remove in | Why wait |
|---|-----------|-----------|----------|
| B1 | `swsssdk-py3` | 202705 | Still imported at runtime (`dualtor_neighbor_check.py`), and by `sonic-py-common` and the vpp container. Move them to `swsscommon` first. |
| B2 | Kubernetes worker (`INCLUDE_KUBERNETES`) | 202711 | Off by default, but still patched upstream, so users may exist. Deprecate and see. **Keep the ctrmgrd wrapper — it is not k8s-only.** |

#### 7.3 Looked at, keeping

We checked these and are **not** proposing removal. Listed so reviewers know we looked.

- **Live silicon vendors:** centec, centec-arm64, marvell-teralynx, pensando, nokia-vs. Out of CI does not mean unused — these have real HWSKUs or recent feature work.
- **Real (if niche) users:** `docker-sonic-p4rt` (Google PINS), `docker-iccpd` (MCLAG), `docker-sonic-redfish` (new in 2026, aspeed BMC, in CI).
- **Test-only, not image surface:** `docker-ptf`, `docker-ptf-sai`.
- **p4lang toolchain** (`rules/p4lang.mk`): kept — DASH SAI and PTF depend on it.

### 8. Per-candidate detail

Short notes on the ones with real gaps or tricky scope.

**A1 — Barefoot / DTEL.** Tofino is EOL. The platform is absent from the CI build matrix (aspeed_arm64, broadcom, marvell-prestera, mellanox, nvidia_bluefield, vs, alpinevs, vpp) — so it is unbuilt and untested. A stale `barefoot` job still sits in `azure-pipelines-build.yml` and should go too. DTEL (`dtelorch`) only ran on Tofino in practice, and TAM (202605) covers the same use case, so DTEL goes with it.

**A2 — P4 platform.** This is the old bmv2 software switch (`docker-sonic-p4`, `sai-p4-bm`, `p4c-bm`, `tenjin`, `p4-hlir`), not the p4lang toolchain. The only reference outside `platform/p4/` is a stale Debian-8 TODO in `rules/redis.mk`. Remove `platform/p4/*`, its 4 submodules, and that TODO. **Do not touch `rules/p4lang.mk`** — DASH SAI (in `sonic-vs.img.gz` via `INCLUDE_VS_DASH_SAI`) and the PTF containers still need bmv2/p4c/PI.

**A8 — System Telemetry.** The telemetry and gnmi containers both come from `sonic-gnmi`. `telemetry.sh` and `gnmi-native.sh` read the same `TELEMETRY|gnmi` config table; the gnmi one just adds more (ZMQ, VRF binding). Dial-out exists in both. So there is near-zero config migration. Gap: confirm `gnmi_server` fully matches the old `telemetry` binary before deleting.

**A9 — FRR modes.** `docker_init.sh` has four modes. `separated` and `split` write per-daemon files (`bgpd.conf`, `zebra.conf`, ...). FRR 10.5 no longer reads those, so these two modes are broken. The working path uses one unified `frr.conf` (`service integrated-vtysh-config`). This is really a dead-code fix. Open question: what the default should become — see [12](#12-open-items).

**A10 — old Debian bases.** Two shapes. **stretch** and **buster** each have real `docker-base-*` / `docker-config-engine-*` / `docker-swss-layer-*` containers to delete. **jessie** does not — there is no `docker-base-jessie`. What's left of jessie is the `sonic-slave-jessie` build slave and its `make jessie` target (Debian 8), plus jessie strings in the generic `docker-base` (armhf/arm64 sources, `FROM ...:jessie`). Drop the jessie slave and its wiring; check the generic `docker-base` before touching it. Many other jessie strings sit in code we already remove (p4, nephos). Keep bullseye — it may still be a live base.

**B1 — swsssdk-py3.** Looks dead but isn't. `swsscommon` replaces it, but real code still imports `swsssdk` at runtime. Migrate those, then drop the wheel.

### 9. Config and management impact

- **A6 / A8 / B2:** these are feature containers. Removing them drops their `FEATURE` table entries and build flags. No CLI change for users who never turned them on.
- **A9:** removes two values from `docker_routing_config_mode`. The surviving default is an open item.
- **A2 / A3 / A1:** platform removals. No effect on other platforms.
- No YANG changes beyond dropping models for removed features (e.g. `sonic-kubernetes_master.yang` with A6).

### 10. Warmboot/Fastboot impact

None. Every candidate is off by default, dead, or platform-specific to hardware being removed. Removing them does not touch the warmboot/fastboot path for supported platforms.

### 11. Testing

- **Build:** each supported CI platform still builds after removal.
- **Image diff:** confirm removed packages are gone from the SBOM and CVE scan.
- **A8:** gnmi container still serves Get/Set/Subscribe and dial-out.
- **A9:** default FRR path still comes up and programs routes.
- **B1:** `dualtor_neighbor_check` and `sonic-py-common` work on `swsscommon`.
- **A10:** all remaining containers build with nothing pointing at the deleted bases.

### 12. Open items

Held back until an owner confirms. Not proposed here.

- **FRR default mode (A9):** which single mode survives, and whether the FRR 10.5 upgrade already moved the default. Needs routing maintainers.
- **REST API** (`docker-sonic-restapi`): baremetal VNET config API. gNMI does not cover its bulk-route (207 partial) and route-expiry semantics. Needs Microsoft to confirm no live consumer.
- **Kubernetes worker owner check:** confirm the user base before setting a firm removal date.
- **PDE** (`docker-pde`): Broadcom bring-up tool. Confirm with Broadcom.
- **clounix platform:** nearly inert but added under a year ago. Confirm intent with the vendor.
