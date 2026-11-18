# SONiC-VPP Bare-Metal Platform Enablement and Performance Characterization — High Level Design

Rev v0.1

## Table of Contents

1. [Revisions](#1-revisions)
2. [Scope](#2-scope)
3. [Definitions/Abbreviations](#3-definitionsabbreviations)
4. [Overview](#4-overview)
5. [Bare-Metal Deployment on x86 Servers](#5-bare-metal-deployment-on-x86-servers)
6. [NIC Binding](#6-nic-binding)
7. [Deployment Profiles](#7-deployment-profiles)
8. [Test Plan: Route Reflector Profiling](#8-test-plan-route-reflector-profiling)
9. [Test Plan: L3 Forwarding Throughput per Core](#9-test-plan-l3-forwarding-throughput-per-core)
10. [Change Summary](#10-change-summary)
11. [Phasing](#11-phasing)
12. [Risks and Open Questions](#12-risks-and-open-questions)
13. [References](#13-references)

## 1. Revisions

| Rev  | Date       | Author(s)    | Changes       |
|------|------------|--------------|---------------|
| v0.1 | 17/11/2026 | yue-fred-gao | Initial draft |

## 2. Scope

This document describes bare-metal platform enablement for SONiC-VPP: moving
it from the virtual `x86_64-kvm_x86_64-r0` platform onto physical x86_64
servers. It also covers characterizing its performance in two separate roles. Each role is a separate DUT
configuration and test campaign; they are never run at the same time.

The design is hardware-generic: any x86_64 server that meets the requirements
in [Section 5.1](#51-target-hardware-requirements) is a valid target. The
**first target is a Cisco UCS C-series server**. It is used as the worked
example throughout, and its specifics are collected in
[Section 5.5](#55-reference-target-cisco-ucs-c-series).

- A **pure BGP route reflector (RR)**: BGP best paths are reflected to clients
  but not installed in the forwarding table. Used only for the RR profiling in
  [Section 8](#8-test-plan-route-reflector-profiling).
- A **regular L3 router** (no RR configuration, routes installed in the VPP
  FIB), to measure forwarding throughput as a function of the number of VPP
  worker cores in [Section 9](#9-test-plan-l3-forwarding-throughput-per-core).

In scope:

- Installing the `sonic-vpp.bin` (ONIE) or `sonic-vpp.img.gz` / `sonic-vpp.raw`
  (disk image) build outputs on an x86_64 server, with Cisco UCS as the first
  target.
- A generic bare-metal platform, with one HWSKU per server model and NIC
  combination.
- Binding physical NICs to VPP through DPDK, with NUMA-aware CPU, queue and
  memory configuration.
- Two deployment profiles, `rr` and `l3-router`, each covering FRR/BGP, SONiC
  system and VPP/host configuration.
- Test plans for RR scale and update propagation, and for forwarding throughput
  versus core count, using Ixia (Keysight IxNetwork) or Spirent as the tester.
- Published performance numbers for the first UCS HWSKU with each profile.

Out of scope:

- RR with routes installed into the VPP FIB ("RR + router"). The forwarding
  test in [Section 9](#9-test-plan-l3-forwarding-throughput-per-core) covers
  FIB scale separately.
- Production ONIE support for servers. ONIE is used here as an install vehicle
  only.
- Vendor platform APIs (sensors, fans, PSUs, LEDs through the BMC). The generic
  platform runs without them.
- Non-x86 servers (Arm).
- Warm reboot, multi-ASIC, and high availability of the RR.

## 3. Definitions/Abbreviations

| Term | Definition |
|------|------------|
| RR | BGP route reflector (RFC 4456) |
| Client | An iBGP peer configured as `route-reflector-client` on the RR |
| BMC | Baseboard management controller (Cisco CIMC, Dell iDRAC, HPE iLO, Supermicro IPMI, or any Redfish BMC) |
| Virtual media | BMC feature that mounts an ISO as a boot device |
| LOM | LAN on motherboard: onboard NIC ports |
| HWSKU | SONiC hardware SKU: a directory with the port map and per-SKU settings |
| CIMC | Cisco Integrated Management Controller, the UCS BMC |
| VIC | Cisco Virtual Interface Card (`enic` driver), the UCS converged adapter |
| PMD | DPDK poll mode driver |
| NDR / PDR | No-drop rate / partial-drop rate (here 0.5% loss), from RFC 2544 search |
| EoR | BGP End-of-RIB marker |
| BMP | BGP Monitoring Protocol (RFC 7854, RFC 9069 for Loc-RIB) |
| CoPP | Control-plane policing |

## 4. Overview

The same server and image are used in two DUT modes, one per test campaign.
Each mode is a deployment profile ([Section 7](#7-deployment-profiles)).

**Mode 1 — pure route reflector, `rr` profile** (Section 8). Only BGP crosses
the data ports.

```
   +------------------------- Ixia chassis -------------------------+
   |  port A: route sources (iBGP)     port B..: RR clients (iBGP)  |
   +------|-----------------------------------|---------------------+
          |                                   |
   +------|-----------------------------------|---------------------+
   |   Ethernet0                         Ethernet4..                |
   |   VPP (DPDK) --- punt BGP ---> Linux host ifs ---> bgpd (RR)   |
   |   FRR: route-reflector-client, bgp no-rib (no FIB install)    |
   |   x86 server (first: Cisco UCS)          eth0 = mgmt           |
   +----------------------------------------------------------------+
```

**Mode 2 — regular L3 router, `l3-router` profile** (Section 9). No RR
configuration. Routes from
eBGP peers are installed in the VPP FIB, and Ixia traffic is forwarded by VPP.

```
   +------------------------- Ixia chassis -------------------------+
   |  port X: eBGP peer + traffic        port Y: eBGP peer + traffic|
   +------|-----------------------------------|---------------------+
          |                                   |
   +------|-----------------------------------|---------------------+
   |   Ethernet0  <------- VPP forwarding ------->  Ethernet4       |
   |   bgpd -> zebra -> fpmsyncd -> orchagent -> syncd -> VPP FIB   |
   |   x86 server (first: Cisco UCS)          eth0 = mgmt           |
   +----------------------------------------------------------------+
```

The image is the standard SONiC-VPP build. Today it supports only one platform,
`device/virtual/x86_64-kvm_x86_64-r0` (`platform_asic` = `vpp`, default SKU
`Force10-S6000`). The work in this document is mostly platform files, start-up
scripts and test assets. No VPP data-plane or SAI changes are expected.

## 5. Bare-Metal Deployment on x86 Servers

Sections 5.1–5.4 apply to any x86_64 server. Section 5.5 gives the specifics
for the first target, Cisco UCS C-series.

### 5.1 Target hardware requirements

| Item | Requirement | Recommended for profiling |
|------|-------------|---------------------------|
| CPU | x86_64 with SSE4.2 and AVX2 (Intel Xeon or AMD EPYC) | AVX-512 capable; high base clock (RR is clock-bound, see [Section 7.5](#75-rr-profile-cpu-split)) |
| Sockets / NUMA | 1 or 2 sockets | Data NICs and VPP workers on the same NUMA node |
| Memory | 32 GB | 64 GB+, so 1 GB hugepages and a 1M-route FIB fit alongside FRR and redis |
| IOMMU | Intel VT-d or AMD-Vi, enabled in firmware | — |
| Firmware | UEFI or legacy BIOS; Secure Boot off (or a signed image) | UEFI |
| Boot disk | ≥ 32 GB, visible to the SONiC kernel (SATA, SAS/RAID, NVMe) | NVMe or a RAID-1 virtual drive |
| Management | One NIC with a kernel driver for `eth0`; BMC with virtual media and serial-over-LAN | Redfish-capable BMC, for scripted installs |
| Data NICs | DPDK-supported, see [Section 6.2](#62-nic-support) | Intel E810 in a PCIe Gen4 x16 slot |

### 5.2 Install paths

Most servers do not ship with ONIE. There are two install paths. Both work
through any BMC with virtual media, or through PXE.

**Path A — ONIE, using `sonic-vpp.bin`.**

1. Mount `onie-recovery-x86_64-kvm_x86_64-r0.iso` (already downloaded by
   `platform/vpp/rules/onie.mk`) through BMC virtual media, and boot it.
2. Select "ONIE: Embed ONIE" to install ONIE on the boot disk.
3. From ONIE: `onie-nos-install http://<server>/sonic-vpp.bin`.

ONIE reports `onie_platform=x86_64-kvm_x86_64-r0`. That name is in the image's
`platforms_asic` list, so `installer/install.sh` does not raise the
"different ASIC type" prompt.

Risk: the kvm ONIE kernel targets virtio. On a given server it may lack the
storage controller driver or the onboard NIC driver. If ONIE cannot see the
disk or reach the network, a custom ONIE build with those drivers is needed.
Driver coverage is checked once per server model.

**Path B — disk image, using `sonic-vpp.img.gz` or `sonic-vpp.raw`.**

1. Boot a live Linux (Debian live ISO through virtual media, or PXE).
2. `zcat sonic-vpp.img.gz | dd of=/dev/<boot-disk> bs=4M oflag=direct`.
3. Grow the SONiC partition to fill the disk (`growpart`, `resize2fs`).
4. Set the platform in `/host/machine.conf` (see [Section 5.4](#54-platform-identity)).

The SONiC kernel has the full Debian driver set, so Path B works on almost any
server.

**Recommendation:** use Path B for first bring-up on each new server model,
since it carries the least driver risk. Use Path A as the repeatable path once
validated, because it also supports `sonic-installer` upgrades in the normal
way.

### 5.3 Server preparation

The settings are the same on every server; only where to set them differs by
vendor (BIOS token names, BMC menus).

| Area | Setting |
|------|---------|
| Firmware | IOMMU on (Intel VT-d / AMD-Vi); SR-IOV on if VFs are used; Secure Boot off (or sign the image) |
| Firmware (perf) | Performance power profile, C-states off, fixed turbo setting; record the NUMA split mode (Intel SNC / AMD NPS) |
| Boot | Boot disk first; UEFI preferred |
| Console | BMC serial-over-LAN on `ttyS0` or `ttyS1` (vendor-dependent), 115200; GRUB and kernel console set to match |
| Kernel cmdline (all) | `iommu=pt` plus `intel_iommu=on` on Intel (AMD-Vi is on by default) |
| Kernel cmdline (per HWSKU) | `default_hugepagesz=1G hugepagesz=1G hugepages=<N>`, and for profiling `isolcpus`, `nohz_full`, `rcu_nocbs` on the worker cores |
| Management | The chosen onboard port pinned to `eth0` with a udev rule, matched by PCI address |

The common arguments go in the platform's `installer.conf`
(`ONIE_PLATFORM_EXTRA_CMDLINE_LINUX`). Today
`device/virtual/x86_64-kvm_x86_64-r0/installer.conf` sets only
`swiotlb=65536`. Hugepage count and isolated cores depend on the server's memory
and core layout. They come from the HWSKU and are written into the GRUB
config by a host-tuning step on first boot; a reboot applies them.

### 5.4 Platform identity

Running as `x86_64-kvm_x86_64-r0` is acceptable for bring-up. For repeatable
bare-metal use, add **one generic platform** for all servers, and **one HWSKU
per server model and NIC layout**. A vendor-specific platform directory is only
needed later, if a vendor's platform APIs are integrated.

```
device/generic/x86_64-vpp_server-r0/
  platform_asic                    # "vpp", so the image lists this platform
  installer.conf                   # common kernel cmdline (Section 5.3)
  pmon_daemon_control.json         # skip xcvrd, psud, thermalctld, ledd, ...
  default_sku                      # e.g. "UCS-C220M7-E810-2x100G t1"
  profiles/                        # role profiles: rr, l3-router (Section 7)
  UCS-C220M7-E810-2x100G/          # first HWSKU (Section 5.5)
    port_config.ini                # real data ports and speeds
    sonic_vpp_ifmap.ini            # Ethernet<N> -> VPP interface name
    vpp_server.conf                # PCI allowlist, mgmt PCI address,
                                   # hugepages, core lists (Section 6.3)
    profiles/                      # optional per-HWSKU profile overrides
  <Vendor>-<Model>-<NIC>-<ports>/  # further servers: add a HWSKU only
```

`vpp_server.conf` holds everything that differs between servers: the data-port
PCI allowlist, the management NIC PCI address, the hugepage count, and the VPP
main and worker core lists. Adding a new server model means adding a HWSKU
directory and running the inventory helper from
[Section 6.3](#63-design) to fill it in. No script changes are needed.

The platform name is set at install time. A future server ONIE build would
report `x86_64-vpp_server-r0`; with the kvm ONIE (Path A) or the disk image
(Path B), `/host/machine.conf` is edited to set it, and the HWSKU is chosen with
`default_sku` or `config-setup`.

### 5.5 Reference target: Cisco UCS C-series

The first target is a UCS C-series rack server (for example C220 or C240,
M6 or M7). The exact model and NIC are not decided yet. The first HWSKU is named
for them when known; `UCS-C220M7-E810-2x100G` is used here as a placeholder.

| Topic | UCS specifics |
|-------|---------------|
| BMC | CIMC. Virtual media via the KVM console or the CIMC "vMedia" mappings; also scriptable through CIMC's Redfish API |
| ONIE | Not shipped. Use Path A with the kvm ONIE ISO, or Path B |
| Storage | Cisco 12G SAS RAID (MegaRAID, `megaraid_sas`), HBA (`mpt3sas`), or NVMe. Check that kvm ONIE sees the disk before relying on Path A |
| Console | CIMC serial-over-LAN, usually `ttyS0`; set the baud rate to match in CIMC and GRUB |
| Firmware tokens | "Intel VT for Directed IO" on; "CPU Performance" Enterprise/HPC; "Package C State Limit" C0/C1; "Processor C6 Report" off; record "SNC" |
| Management NIC | An onboard LOM port, or CIMC shared-LOM mode. Pin it to `eth0` by PCI address |
| VIC adapter | Cisco VIC 1400/1500 (`enic`). Keep it for management, or for data only if no other NIC is fitted (see [Section 6.2](#62-nic-support)). VIC vNICs must not be picked up as data ports |
| Data NIC | A PCIe NIC (Intel E810 recommended) in a riser slot wired to the CPU socket that will run the VPP workers |

## 6. NIC Binding

### 6.1 Current behavior

At boot:

- `10-01-vpp-cfg-init` walks `eth1..eth32` and adds every interface that has a
  PCI bus address to `VPP_DPDK_PORTS` in `/etc/sonic/vpp/syncd_vpp_env`.
- `vpp-port-config.service` runs `vpp_ports_setup.sh start`, which binds those
  devices to `UIO_DRV` (default `uio_pci_generic`). Devices on `mlx5_core` are
  skipped.
- `vpp_init.sh` in `syncd` writes one `dpdk { dev <pci> { name bobmN } }` per
  port, sets `buffers-per-numa = PERPORT_BUF * SONIC_NUM_PORTS`, sets
  `workers` from `VPP_WORKERS`, and reserves about 12 × 2 MB hugepages.

On a server this has five problems:

1. Every PCI NIC after `eth0` is taken as a data port: spare LOM ports, and
   adapters such as UCS VIC vNICs.
2. `uio_pci_generic` does not support MSI-X and is blocked under Secure Boot.
3. 2 MB hugepages in small numbers are not enough for 100G ports and many
   buffers.
4. Workers are auto-pinned from `main-core 2` with `skip-cores 1`, regardless
   of which NUMA node the NIC is on.
5. Every port gets the default single RX queue, so RSS never spreads traffic
   across workers.

### 6.2 NIC support

| NIC | DPDK PMD | Status in this build | Use |
|-----|----------|----------------------|-----|
| Intel E810 (25/100G) | `ice` | Supported | Recommended data NIC |
| Intel X710 / XXV710 (10/25G) | `i40e` | Supported | Data NIC |
| NVIDIA ConnectX-5/6/7 | `mlx5` | **Not supported**: the PMD is disabled in `vppbld/Makefile` (`DPDK_MLX5_PMD=n`) and `rdma_plugin` is not enabled in `startup.conf.tmpl` | Needs a separate work item |
| Cisco VIC 1400/1500 (UCS only) | `enic` | Supported by DPDK; needs vNIC queue/interrupt config in CIMC | Management; data only if nothing else |

These are standard PCIe NICs that work the same in any server. Server-specific
adapters (here the UCS VIC) are listed separately. The NIC model for the first
target is not decided yet. The design assumes Intel E810 and keeps the config
generic across PMDs.

### 6.3 Design

1. **Port allowlist.** A new `VPP_PCI_ALLOWLIST` (comma-separated PCI
   addresses) in the HWSKU's `vpp_server.conf`
   ([Section 5.4](#54-platform-identity)), or as an environment override.
   When it is set, `10-01-vpp-cfg-init` uses it instead of scanning `eth1..32`.
   Ports are mapped in list order to `Ethernet0`, `Ethernet4`, and so on, as
   today.
2. **`vfio-pci` on bare metal.** `vpp_ports_setup.sh` defaults to `vfio-pci`
   when `/sys/kernel/iommu_groups` is populated, and to `uio_pci_generic`
   otherwise (the VM case).
3. **Hugepages.** Use the 1 GB pages from the kernel command line. Keep the
   2 MB sizing as the fallback when no 1 GB pages exist.
4. **NUMA-aware CPU and queues.** New `syncd_vpp_env` variables, with defaults
   from `vpp_server.conf`, validated the same way `VPP_WORKERS` is:
   - `VPP_MAIN_CORE` and `VPP_CORELIST_WORKERS`, which replace `skip-cores` /
     `workers` when set.
   - `VPP_RX_QUEUES` (default: number of workers) and `VPP_RX_DESC` /
     `VPP_TX_DESC`, written into `dpdk { dev default { ... } }`.
   - Buffers sized per NUMA node from port count × queues × descriptors.
5. **Memory for scale.** `VPP_MAIN_HEAP` and the `l3fib` pool sizes become
   overridable. `startup.conf.tmpl` is sized for 100K routes, and the
   forwarding tests go to 1M.
6. **Inventory helper** (`vpp_nic_inventory.sh`). For each NIC, prints the PCI
   address, vendor:device ID, NUMA node, current driver, PCIe link
   speed/width, and the CPUs local to that NUMA node. It drafts a
   `vpp_server.conf` for a new HWSKU, so the same tool onboards any server
   model.

### 6.4 Validation

- `vppctl show hardware-interfaces`: PMD driver, link up, negotiated speed,
  RX/TX queue count, NUMA node.
- `vppctl show interface rx-placement`: each queue on a worker on the NIC's
  NUMA node.
- `vppctl show buffers`: no allocation failures.
- SONiC: `show interfaces status` lists each data port as `up`, with the
  correct speed.
- A ping and a simple BGP session across each data port.

## 7. Deployment Profiles

A **profile** is the complete configuration for one role, applied on top of a
HWSKU. It covers three layers: FRR/BGP, SONiC system configuration, and
VPP/host sizing. There are two profiles:

- **`rr`**: pure route reflector (Mode 1).
- **`l3-router`**: regular L3 router (Mode 2).

The performance tests run on these profiles unchanged. Test assets only add
measurement overlays (packet capture, a BMP target, per-sweep overrides such as
the worker count). The published numbers therefore describe a configuration
that users can deploy as-is.

### 7.1 Layout and application

Profiles live in the generic platform directory, next to the HWSKUs
([Section 5.4](#54-platform-identity)):

```
device/generic/x86_64-vpp_server-r0/
  profiles/
    rr/
      profile.conf          # VPP and host sizing rules for the role
      config_db.json.j2     # SONiC system config fragment
      frr.conf.j2           # FRR config (split mode)
      sysctl.conf           # kernel network tuning
      VERSION
    l3-router/
      (same files)
  UCS-C220M7-E810-2x100G/
    vpp_server.conf         # hardware facts: PCI allowlist, mgmt NIC, NUMA CPUs
    profiles/
      rr/profile.conf       # optional per-HWSKU overrides (exact core lists,
      l3-router/profile.conf  # hugepage count)
```

Settings are layered: role profile, then the per-HWSKU override, then an
environment override. The environment override is only used by test sweeps.
Role profiles express sizing as rules (for example "all physical cores on
the data NICs' NUMA node, minus the main core and one housekeeping core"). The
rules are resolved against `vpp_server.conf`, so a role profile works on any
HWSKU without per-server copies.

A new script, `vpp_profile apply <rr|l3-router>`, applies a profile:

1. Merges `profile.conf` with `vpp_server.conf`. Writes the VPP variables into
   `syncd_vpp_env` and the kernel arguments (hugepages, isolated cores) into
   GRUB through `vpp_host_tune.sh`.
2. Renders `config_db.json.j2` with `sonic-cfggen` and applies it to
   `config_db`.
3. Renders `frr.conf.j2` into `/etc/sonic/frr/` and sets
   `docker_routing_config_mode` to `split`.
4. Installs `sysctl.conf`.
5. Records the profile name and version in `/etc/sonic/vpp/profile`, so every
   test result and support dump shows which profile was active.
6. Reboots if kernel arguments changed. Otherwise it does a config reload.

The profile state lives in files rather than a new `DEVICE_METADATA` field, so
no YANG change is needed.

### 7.2 Profile contents

Initial values are below. Final values are set from the test results in
the platform qualification phase ([Section 11](#11-phasing)) and recorded under a new
profile `VERSION`.

| Layer | Item | `rr` | `l3-router` |
|-------|------|------|-------------|
| VPP | Workers | 1 | All physical cores on the NIC NUMA node minus main and housekeeping |
| VPP | RX queues per port | 1 | Equal to workers |
| VPP | Main heap and `l3fib` pools | Template defaults (FIB stays small) | Sized for 1M routes: main heap ≥ 4 GB, pools ≥ 2M entries |
| Host | 1 GB hugepages | 2 | Sized for buffers plus a 1M-route FIB (initially 8) |
| Host | Isolated cores | None. Cores stay available to bgpd, zebra and redis | VPP worker cores |
| Host | sysctl | Larger TCP socket buffers and backlog for hundreds of BGP sessions | Default |
| SONiC | CoPP | Dedicated BGP trap group with raised CIR/CBS ([Section 7.4](#74-rr-profile-control-plane-policing)) | SONiC default |
| SONiC | Features | Unneeded containers off (for example `dhcp_relay`, `sflow`) | Default |
| SONiC | `docker_routing_config_mode` | `split` | `split` |
| FRR | BGP | RR template ([Section 7.3](#73-rr-profile-frr-configuration)) | eBGP template: neighbor peer-group, `maximum-paths` for ECMP, FIB install on |
| FRR | Logging | `log file`, `log timestamp precision 6`, `bgp log-neighbor-changes` | Same |

Neighbor addresses, ASN and router ID come from `config_db`
(`DEVICE_METADATA`, `LOOPBACK_INTERFACE`), and are rendered into
`frr.conf.j2`. The profile contains no site-specific values.

### 7.3 RR profile: FRR configuration

The RR must not push reflected routes through
`zebra → fpmsyncd → APPL_DB → orchagent → syncd → VPP`. FRR 10.7 (the version
in `rules/frr.mk`) supports:

```
bgp no-rib
!
router bgp 65000
 bgp router-id 10.255.0.1
 bgp cluster-id 10.255.0.1
 no bgp default ipv4-unicast
 bgp listen range 10.1.0.0/16 peer-group RRC
 neighbor RRC peer-group
 neighbor RRC remote-as 65000
 !
 address-family ipv4 unicast
  neighbor RRC activate
  neighbor RRC route-reflector-client
 exit-address-family
 address-family ipv6 unicast
  neighbor RRC activate
  neighbor RRC route-reflector-client
 exit-address-family
```

`bgp no-rib` stops bgpd from installing any BGP route into zebra. A deny
`table-map` per address family is the fallback if `no-rib` interferes with
next-hop tracking. Dynamic neighbors (`bgp listen range`) make it practical to
scale to hundreds of clients without one config stanza per peer.

The `rr` profile uses `split` config mode (`docker_routing_config_mode`), with
this config rendered from `frr.conf.j2`. Mapping the same config onto
`config_db` (`BGP_PEER_RANGE`, handled by bgpcfgd) is a follow-up.

### 7.4 RR profile: control-plane policing

BGP packets arriving on data ports are punted from VPP to the Linux host
interfaces. The BGP trap is policed **inside VPP**:

- The policer is bound by `SwitchVppHostifTrap.cpp` through
  `vpp_sonic_ext_copp_ip2me_bind_condition(..., 179, ...)`.
- It uses `COPP_GROUP.queue4_group1` from `files/image_config/copp/copp_cfg.j2`:
  **6000 pps CIR**, and the trap group is shared with LACP.

At RR scale, this policer limits both incoming UPDATEs and the TCP ACKs coming
back from hundreds of clients. The `rr` profile's `config_db.json.j2`
therefore moves BGP to a dedicated trap group with a raised CIR and CBS. The
initial value is chosen in tester calibration; the final value comes from the
test results. Drops are visible in VPP policer and error counters, and must be
zero in every reported run.

### 7.5 RR profile: CPU split

In pure-RR mode the data plane carries only BGP. VPP runs with
`VPP_WORKERS=0` or `1`, which leaves most cores to bgpd, zebra and redis. bgpd
is mostly single-threaded (main thread plus I/O and keepalive pthreads), so
**per-core clock speed matters more than core count** for RR performance.
Record the CPU model and frequency for every run.

### 7.6 L3 router profile

The `l3-router` profile has no RR settings: no `route-reflector-client` and no
`bgp no-rib`. Routes go through the normal install path
`bgpd → zebra → fpmsyncd → orchagent → syncd → VPP`. It gives VPP the data
NICs' NUMA-local cores, one RX queue per worker on every port, and the memory
needed for a 1M-route FIB. The memory overrides come from
[Section 6.3](#63-design). The throughput tests override only the worker count
(and with it the RX queue count) per sweep step, through the environment layer.

## 8. Test Plan: Route Reflector Profiling

### 8.1 Topology

- DUT: the target server (first: UCS HWSKU) running SONiC-VPP in Mode 1, with
  the `rr` profile applied unchanged
  ([Section 7](#7-deployment-profiles)). No data traffic is forwarded in this
  campaign. Measurement overlays (capture, BMP target) are added by the test
  assets and are not part of the profile.
- Tester: an IxNetwork BGP emulation on 2–4 Ixia ports connected to DUT data
  ports.
  - **Source port(s)**: 1–4 emulated iBGP clients that advertise the route
    ranges.
  - **Client port(s)**: N emulated iBGP clients that advertise nothing and
    receive the reflected routes. Each port carries many peers, using
    per-peer IPs within `10.1.0.0/16`.
- Ixia generates the BGP load and provides per-peer counters (sessions up,
  routes received) for coarse checks and pass/fail. **Timing comes from the
  DUT** ([Section 8.3](#83-timing-sources)). Ixia statistics are polled every
  few seconds, and protocol-packet capture timestamps on Ixia ports depend on
  the load module.

### 8.2 Parameters

A full cross product of all parameters gives more than 1,000 points. Instead,
the plan uses one **baseline** profile, varies **one parameter at a time**
from it, and adds a single **stress** point. This gives 13 parameter points in
total.

**Baseline (B):** 100 clients, 1M IPv4 prefixes, no IPv6, Internet-like prefix
distribution (from a RIPE RIS MRT dump, replayed by Ixia), 1 path per prefix,
all clients in one update group, peering over data ports through VPP.

| Sweep | Varied parameter | Values (baseline in bold) | New points |
|-------|------------------|---------------------------|------------|
| S1 | Clients | 1, 10, **100**, 250, 500 | 4 |
| S2 | IPv4 prefixes | 100K, 500K, **1M** | 2 |
| S3 | Address family | **IPv4 only**; IPv4 1M + IPv6 200K | 1 |
| S4 | Paths per prefix (sources) | **1**, 2, 4 | 2 |
| S5 | Update groups (distinct outbound policies) | **1**, 10 | 1 |
| S6 | Prefix distribution | **Internet-like**; synthetic fixed /24 | 1 |
| X | Stress: 500 clients, 1M IPv4 + 200K IPv6, 4 paths, 10 update groups | — | 1 |

Baseline (1) + sweeps (11) + stress (1) = 13 points.

**Interactions.** If a sweep shows a knee (for example, convergence time
growing faster than linearly with clients), add a 2 × 2 interaction check
between that parameter and the next most influential one, at two levels each.
This adds at most 3 points per interaction, and is only done where the data
shows a need.

**Tests per point.** Not every test runs at every point:

| Test | Points | Runs |
|------|--------|------|
| T1 full-table convergence | All 13 | 3 repeats each |
| T1b new-client sync | B only, K = 1, 10, 100 | 3 repeats each |
| T2 single-update latency | B, S1 (clients), S5 (update groups) | 100 events per point |
| T3 batch propagation | B, S1 | 1K / 10K / 100K batches, 3 repeats |
| T4 sustained churn | B, X | Rate search at 1K / 10K / 50K updates/s |
| T5 disruption and recovery | B | Each scenario 3 times |
| T6 resource profile | Collected during every run | — |

This is roughly 100 runs per campaign, small enough to automate and repeat on
each new server model or software version.

### 8.3 Timing sources

All timestamps are taken on the DUT, so every measurement uses one clock and
no cross-device synchronization is needed. Three sources are used; BMP
provides two kinds of data with different resolution:

| Source | What it timestamps | Resolution | Used for |
|--------|--------------------|------------|----------|
| DUT packet capture | Every BGP segment entering or leaving bgpd, on the wire side of the Linux stack | µs | Inbound and outbound wire times: T1–T5 |
| FRR log | Session state changes (`%ADJCHANGE ... Up/Down`) | µs | Session Established and Down times |
| BMP route mirroring | Each BGP message as bgpd starts processing it | µs | Time inbound messages spend queued before bgpd handles them |
| BMP route monitoring / Loc-RIB | Adj-RIB-In and best-path entries | **1 s** | Correctness, and coarse T1 progress only |

The resolutions are from FRR's `bgpd/bgp_bmp.c`. Mirroring stamps each message
with `gettimeofday()`. Route monitoring and Loc-RIB use the path's `uptime`,
which is whole seconds. FRR's BMP has no Adj-RIB-Out monitoring, so
outbound times always come from capture.

**DUT packet capture.**

- BGP from the data ports is punted by VPP to the Linux host interfaces
  (`Ethernet<N>`). The `bgp` container uses host networking, so `tcpdump` in
  the host namespace sees every BGP packet that bgpd sends or receives.
- Inbound timestamps are taken after the VPP punt path, and outbound ones
  before VPP transmit. VPP adds microseconds, which is negligible next to
  bgpd's millisecond-scale processing. CoPP drops or queuing inside VPP are not
  visible in the capture, which is why T6 requires zero policer drops.
- Two capture modes keep the volume manageable at 1M prefixes × hundreds of
  clients:
  - **Header-only** (`-s 96`, all BGP peers): TCP payload length and time per
    segment. This is enough to find "last UPDATE byte sent to client X",
    because after convergence only 19-byte KEEPALIVEs follow.
  - **Full payload** (selected peers, or short tests only): decoded with
    `tshark` with TCP reassembly, to find specific prefixes and the EoR marker.
    Segmentation offload on the host interfaces can merge several UPDATEs, or
    the last UPDATE and the EoR, into one large segment. Analysis therefore
    always works on the reassembled stream, not on packet counts.
- `tcpdump` is pinned with `taskset` to a core not used by bgpd or VPP, and
  writes to a ring buffer on local NVMe. Capture overhead is checked once per
  campaign: T1 is run with and without capture and compared on Ixia counters.
  The difference should be within 5%.

**FRR log.**

```
log file /var/log/frr/bgpd.log
log timestamp precision 6
router bgp 65000
 bgp log-neighbor-changes
```

Logging goes to a file rather than syslog, so rsyslog rate limits don't apply.
Per-update debugs (`debug bgp updates`) are not used. They log every prefix and
would dominate bgpd CPU at this scale.

**BMP.** The `bgp` container already loads BMP (`-M bmp` in the
`docker-fpm-frr` supervisord config). A collector (gobmp or pmacct `pmbmpd`)
runs on the DUT's management side, or on the tester host through `eth0`.

```
router bgp 65000
 bmp targets PERF
  bmp connect <collector> port 5000 min-retry 100 max-retry 10000
  bmp mirror
  bmp monitor ipv4 unicast pre-policy
  bmp monitor ipv4 unicast loc-rib
  bmp monitor ipv6 unicast pre-policy
  bmp monitor ipv6 unicast loc-rib
```

- Mirroring is on for T2 and T3 (few messages).
- Route monitoring is on for correctness checks.
- During T1 at full scale, both are off, or checked for overhead the same way
  as capture: BMP doubles bgpd's output work for every inbound route.

**Per-update timeline.** For one prefix advertised by a source peer:

```
t_wire_in    DUT capture: UPDATE segment from source arrives at Ethernet<N>
t_bgpd_in    BMP mirror: bgpd starts processing that UPDATE
t_wire_out_i DUT capture: UPDATE carrying the prefix leaves toward client i
```

- Input queuing = `t_bgpd_in − t_wire_in`.
- bgpd processing plus output = `t_wire_out_i − t_bgpd_in` (best path, update
  group, packet build and write).
- End-to-end = `t_wire_out_i − t_wire_in`.

Best path and output cannot be separated below 1 s with stock FRR. That would
need FRR built with tracepoints (LTTng or USDT), which SONiC's FRR build does
not enable.

### 8.4 Metrics and procedures

**T1 — Full-table convergence, clients first.**
1. Bring up all client sessions, then bring up the source peers with the full
   table.
2. Start: the source session reaches Established (FRR log `%ADJCHANGE`).
   First-update time `t_wire_in` is also recorded from capture.
3. End, per client: the last UPDATE segment sent to that client (header-only
   capture). Completeness is confirmed by FRR's sent-prefix count
   (`show bgp ipv4 unicast summary`, PfxSnt) and by Ixia's received-route
   counter both equalling the expected total.
4. Report the total time (to the last client) and the per-client spread.

EoR is **not** the end marker here. FRR sends EoR to each client right after
that client's session comes up, before the source has advertised anything. The
source's routes then arrive as ordinary UPDATEs with no second EoR.

**T1b — New-client sync, table first.**
1. Bring up the source peers and let the RR learn the full table.
2. Bring up clients: one alone, then K at once (K = 10, 100).
3. Start: client session Established (FRR log). End: the EoR sent to that
   client (full-payload capture on a sample of clients, header-only on the
   rest).
4. Report the time per client and the total for K simultaneous clients.

**T2 — Single-update propagation latency.**
1. With the table converged, advertise one new prefix (then withdraw it) from a
   source peer.
2. Full-payload capture on the DUT plus BMP mirroring. Match the UPDATE by
   prefix across the timeline in [Section 8.3](#83-timing-sources).
3. Report end-to-end latency (`t_wire_out_i − t_wire_in`) as median, 99th
   percentile and max across clients, with the input-queuing split. Repeat 100
   times.

**T3 — Batch propagation.** Same as T2, with batches of 1K, 10K and 100K
prefixes. Report the time from the first prefix in to the last prefix out, at
the last client.

**T4 — Sustained churn.**
1. Use the Ixia route-flap feature at 1K, 10K and 50K prefix updates/s
   (advertise/withdraw).
2. Every 10 s, a source peer also advertises a unique marker prefix. Its
   propagation latency is measured as in T2 (full-payload capture filtered to
   the marker), which gives lag over time under load.
3. Report the highest rate at which the marker latency stays bounded over
   10 minutes, and client receive rate keeps up with source rate (Ixia
   counters).
4. Watch `show bgp update-groups statistics`, peer OutQ, and bgpd CPU.

**T5 — Disruption and recovery.** Session times from the FRR log, update times
from capture.
- Flap one client: time from its Established to its last UPDATE. Other clients
  must receive no UPDATEs.
- Flap a source peer: time from its Down (FRR log) to the last withdrawal sent
  to the last client.
- Restart the `bgp` container and the whole DUT: time to full re-convergence,
  measured as in T1.
- Graceful restart of the source peer: routes retained, no withdrawals sent to
  clients (capture).

**T6 — Resource profile** (collected during every test):
- bgpd, zebra and redis RSS and CPU (sampled at 1 s).
- `show bgp summary` memory, and `show event cpu` (FRR 10).
- VPP punt and policer counters (`show errors`, `show policer`), which must
  show zero CoPP drops.
- Confirmation that `APPL_DB` `ROUTE_TABLE` stays empty (pure RR).
- Capture and BMP overhead checks from [Section 8.3](#83-timing-sources).

### 8.5 Reporting

- **Baseline summary:** one table with every test result at point B (T1, T1b,
  T2 median/99th/max with input-queuing split, T3, T4 maximum sustained rate,
  T5), plus peak bgpd memory and average and peak bgpd CPU.
- **One chart per sweep,** with the varied parameter on the x-axis:
  - S1: T1 convergence time, T2 99th-percentile latency and T3 time vs. client
    count.
  - S2: T1 convergence time and peak bgpd memory vs. prefix count.
  - S3–S6: T1 convergence time and peak bgpd memory against the baseline, as
    a bar chart.
- **Stress point:** pass/fail (all sessions stay up, all clients converge, zero
  CoPP drops), with T1 and T4 results.
- T1 values are the median of 3 repeats, with the min–max range shown.

Every result set is tagged with the HWSKU, profile name and version, image
version, CPU model and frequency, and BIOS power settings, so results from
different server models can be compared.

## 9. Test Plan: L3 Forwarding Throughput per Core

### 9.1 Topology and DUT mode

- The DUT runs in Mode 2, as a regular L3 router (see
  [Section 4](#4-overview)), with the `l3-router` profile applied
  ([Section 7.6](#76-l3-router-profile)). No RR settings are present.
- Each sweep step overrides only the worker count (and RX queue count) through
  the profile's environment layer. In the `rr` profile, by contrast, VPP is
  limited to 1 worker so cores are left to bgpd.
- 2 or 4 Ixia ports are connected back-to-back to DUT data ports on the same
  NIC/NUMA node. Traffic runs bidirectionally between port pairs.
- Routes are loaded by Ixia eBGP peers through the normal SONiC path. The
  install time for each FIB size is recorded as a side metric.

### 9.2 Method

IxNetwork RFC 2544 QuickTest:

- Throughput by binary search for NDR and PDR (0.5% loss), 60 s trials.
- Latency (min/avg/max, plus 99th percentile where available) at 50%, 90% and
  100% of NDR.

The test names and FIB sizes follow FD.io CSIT (`ip4base`, `ip4scale20k`,
`ip4scale200k`, `ip4scale2m`, and the IPv6 equivalents). This allows results
to be checked against published CSIT numbers for similar CPUs and NICs.

### 9.3 Parameters

| Parameter | Values |
|-----------|--------|
| VPP workers | 0 (main thread forwards), 1, 2, 4, 8, 12, 16 — limited to physical cores on the NIC's NUMA node minus main and housekeeping |
| RX queues per port | = number of workers |
| Hyperthreading | Workers on physical cores; then workers on sibling threads (2 per core) |
| Frame size | 64, 128, 256, 512, 1024, 1518, IMIX |
| Address family | IPv4, IPv6 |
| FIB size | 1 route; 20K; 200K; 1M (needs the memory overrides in [Section 6.3](#63-design)) |
| Flows | ≥ 1024 5-tuples per direction, so RSS uses every queue; plus a 1-flow case to show single-queue limits |
| Features | Baseline; then ACL on ingress, sFlow, route flow counters, and `sonic_ext` features toggled off through `SONIC_EXT_CONFIG` |

### 9.4 Metrics

- NDR/PDR in Mpps and Gbps, and Mpps per worker core.
- Latency at the set load points.
- Per-node CPU cycles per packet and vectors per call, from
  `vppctl show runtime` captured at NDR.
- Bottleneck attribution:
  - `show hardware` `rx_missed` / `imissed` with workers below 100% means NIC
    or PCIe bound.
  - Vectors per call near 256 with no NIC misses means CPU bound.

### 9.5 Reporting

- Mpps vs. worker count, one line per frame size, at each FIB size.
- Scaling efficiency = Mpps(N) / (N × Mpps(1)).
- A cycles-per-packet breakdown by node at 1 worker and at the maximum worker
  count.
- The HWSKU, profile name and version, image version, and host settings (BIOS,
  kernel command line, NIC firmware, CPU model and frequency) are recorded with
  every result set, so per-core numbers from different servers can be
  compared.

### 9.6 Publishing

Results from Sections 8 and 9 are published per HWSKU in
`platform/vpp/docs/perf/<HWSKU>.md`. Each report covers:

- The hardware: server model, CPU, memory, NIC and firmware, slot and NUMA
  placement.
- The image version, and the profile name and version used for each campaign.
- The RR results ([Section 8.5](#85-reporting)) and forwarding results
  ([Section 9.5](#95-reporting)).
- Known limits: which bottleneck capped each curve (line rate, NIC, PCIe, CPU,
  tester).

The report is regenerated for each new profile version or major image
release. The scripts that produce the charts are kept in
`platform/vpp/tests/perf/`.

## 10. Change Summary

| Area | File(s) | Change |
|------|---------|--------|
| Port selection | `platform/vpp/sonic-platform-modules-vpp/files/etc/config-setup/factory-default-hooks.d/10-01-vpp-cfg-init` | Read the HWSKU's `vpp_server.conf`; `VPP_PCI_ALLOWLIST`; write the new CPU/queue/memory variables to `syncd_vpp_env` |
| Host tuning | `platform/vpp/sonic-platform-modules-vpp/files/usr/bin/vpp_host_tune.sh` (new) | Write per-HWSKU hugepages and isolated cores into the GRUB config on first boot |
| Driver binding | `platform/vpp/sonic-platform-modules-vpp/files/usr/bin/vpp_ports_setup.sh` | Default to `vfio-pci` when an IOMMU is present |
| VPP config generation | `platform/vpp/docker-syncd-vpp/scripts/vpp_init.sh` (and the `docker-sonic-vpp` copy) | Core list, RX queues and descriptors, per-NUMA buffers, 1 GB hugepages, heap and `l3fib` overrides |
| Inventory | `platform/vpp/sonic-platform-modules-vpp/files/usr/bin/vpp_nic_inventory.sh` (new) | NIC / NUMA / driver report; drafts `vpp_server.conf` for a new HWSKU |
| Generic platform | `device/generic/x86_64-vpp_server-r0/` (new) | `platform_asic`, `installer.conf`, `pmon_daemon_control.json`, `default_sku`, udev rule for management naming |
| First HWSKU | `device/generic/x86_64-vpp_server-r0/UCS-<model>-<NIC>-<ports>/` (new) | `port_config.ini`, `sonic_vpp_ifmap.ini`, `vpp_server.conf`, optional `profiles/` overrides for the UCS target |
| Profiles | `device/generic/x86_64-vpp_server-r0/profiles/{rr,l3-router}/` (new) | `profile.conf`, `config_db.json.j2`, `frr.conf.j2`, `sysctl.conf`, `VERSION` |
| Profile tool | `platform/vpp/sonic-platform-modules-vpp/files/usr/bin/vpp_profile` (new) | Merge profile layers and apply them to `syncd_vpp_env`, GRUB, `config_db`, FRR and sysctl; record the active profile |
| Test assets | `platform/vpp/tests/perf/` (new) | IxNetwork configs; measurement overlays only (BMP target, capture wrapper with pinned `tcpdump` in header-only and full modes, per-sweep overrides); analysis scripts that merge capture, FRR log and BMP into per-update timelines; resource collection (FRR, VPP); report generation |
| Docs | `platform/vpp/docs/README.baremetal.md` (new) | Generic install and bring-up guide, with a UCS walkthrough, profile usage, and an "adding a new server model" section |
| Performance report | `platform/vpp/docs/perf/<HWSKU>.md` (new) | Published RR and forwarding numbers for the UCS HWSKU ([Section 9.6](#96-publishing)) |
| Later | `platform/vpp/vppbld/Makefile`, `startup.conf.tmpl` | Enable `mlx5` PMD / `rdma_plugin` if NVIDIA NICs are chosen |

## 11. Phasing

Platform work sits on either side of performance characterization. Phase 2
(platform enablement) creates the HWSKU and profiles first, so every config made
for the tests has a permanent home. Phase 5 (platform qualification) then folds
the test results back into the profiles and publishes the numbers.

1. **Bring-up on UCS.** Path B install on the kvm platform name, NIC inventory,
   and the script changes from [Section 6.3](#63-design): allowlist,
   `vfio-pci`, 1 GB hugepages, core list and RX queue variables. Exit: data
   ports up in SONiC, and BGP established across a data port.
2. **Platform enablement: HWSKU and profiles.**
   - Generic platform `x86_64-vpp_server-r0` and the first UCS HWSKU
     (`port_config.ini`, `sonic_vpp_ifmap.ini`, `vpp_server.conf`), based on
     the Phase 1 inventory.
   - `vpp_profile` tool and `vpp_host_tune.sh`.
   - `rr` and `l3-router` profiles v0, with the initial values from
     [Section 7.2](#72-profile-contents): FRR, CoPP, features, sysctl, VPP and
     host sizing.

   Exit: a fresh install plus `vpp_profile apply <profile>` reproduces each DUT
   configuration with no manual steps.
3. **Performance characterization: `l3-router` profile.** The
   [Section 9](#9-test-plan-l3-forwarding-throughput-per-core) matrix,
   including 1M-route FIB scale, and the plain-VPP comparison.
4. **Performance characterization: `rr` profile.** Tester calibration (including the initial CoPP
   value), then the [Section 8](#8-test-plan-route-reflector-profiling)
   points and T1–T6.
5. **Platform qualification: finalize profiles and publish.**
   - Fold the tuned values back into the profiles: CoPP rate, worker and queue
     counts, hugepages and heap, FRR settings. Bump the profile `VERSION`.
   - Rerun the baseline points (RR point B, forwarding at 64-byte and IMIX
     frames) on the final profiles to confirm.
   - Path A (ONIE) install on UCS, the bring-up guide, and the performance
     report `platform/vpp/docs/perf/<HWSKU>.md`
     ([Section 9.6](#96-publishing)).

## 12. Risks and Open Questions

- **NIC model not decided.** If NVIDIA ConnectX is chosen, `mlx5` /
  `rdma_plugin` enablement becomes a Phase 1 dependency.
- **kvm ONIE driver coverage** varies by server model (storage controller,
  management NIC); for UCS, the RAID controller is the main concern. Fallback:
  Path B, or a custom ONIE build.
- **Generic platform vs. vendor platforms.** One generic platform keeps
  onboarding cheap but gives no sensors, fans or PSU data. If that becomes a
  requirement, add vendor platform directories that reuse the same HWSKU
  layout.
- **Serial console and BIOS differences** between vendors (`ttyS0` vs.
  `ttyS1`, token names). These are captured per HWSKU and in the bring-up
  guide, not in scripts.
- **CoPP policer.** The value used for RR testing must be justified, and a
  production value chosen. Check whether ACKs from passive-side sessions
  (where the RR's port is not 179) match the BGP trap condition.
- **Tester limits.** Ixia per-port peer and route capacity, and how fast
  emulated peers absorb UPDATEs, may cap the higher test points.
- **Measurement overhead.** Packet capture and BMP run on the DUT and compete
  with bgpd. Mitigation: dedicated cores, header-only capture at scale, BMP
  off during full-scale T1, and with/without comparisons on Ixia counters.
- **BMP timestamp resolution.** Stock FRR fills route-monitoring and Loc-RIB
  timestamps in whole seconds, so best-path time cannot be isolated at
  sub-second resolution. If needed, build FRR with tracepoints for profiling
  runs only.
- **`bgp no-rib` interactions** with next-hop tracking and with SONiC
  bgpcfgd templates. The fallback is a `table-map` deny.
- **Interface naming on bare metal.** Kernel `ethN` order follows PCI
  enumeration. Without the udev rule, `eth0` may not be the management port.
- **Hyperthreading and SNC** settings change the per-core numbers
  significantly. Fix them per test campaign and report them.

## 13. References

- [RFC 4456 — BGP Route Reflection](https://www.rfc-editor.org/rfc/rfc4456)
- [RFC 2544 — Benchmarking Methodology for Network Interconnect Devices](https://www.rfc-editor.org/rfc/rfc2544)
- [RFC 7854 — BMP](https://www.rfc-editor.org/rfc/rfc7854), [RFC 9069 — BMP Loc-RIB](https://www.rfc-editor.org/rfc/rfc9069)
- [FRR BGP documentation](https://docs.frrouting.org/en/latest/bgp.html)
- [FD.io CSIT reports](https://csit.fd.io/)
- [ONIE](https://opencomputeproject.github.io/onie/)
- [SONiC-VPP HLD](SONICVPP-HLD.md)
