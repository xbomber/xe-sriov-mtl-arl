# Experimental Xe SR-IOV patches for Meteor Lake / Arrow Lake

Independent research into the missing Xe PF services used by Intel Windows SR-IOV guests on MTL/ARL. **Work in progress: rendering is still incomplete.**

Read the [disclaimer](DISCLAIMER.md), [known issues and tests](docs/TESTING.md), and [development provenance](docs/PROVENANCE.md) before using the patch.

## Exact kernel base

| Item | Version |
| --- | --- |
| Source repository | [intel/mainline-tracking](https://github.com/intel/mainline-tracking) |
| Kernel version | **7.2.7** |
| Base commit | [`c4227406af3b9ad8f8072b222ca06e0039f64441`](https://github.com/intel/mainline-tracking/commit/c4227406af3b9ad8f8072b222ca06e0039f64441) |
| Source tag | `mainline-preprod-v7.2.7-linux-260925T043548Z` |
| Tested local kernel release | `7.2.7-adv-xe-ggtt-ctb+` |
| Patch | [linux-7.2.7-xe-mtl-arl-sriov-experimental.patch](patches/linux-7.2.7-xe-mtl-arl-sriov-experimental.patch) |

The patch is extracted from the source used to build the tested kernel. It changes 20 Xe files (620 insertions, 2 deletions). Applicability to other commits, vanilla Linux, or Ubuntu distribution kernels has not been established.

## What the patch implements

- Enables the SR-IOV capability flag on the MTL descriptor shared with ARL.
- Adds the VF-to-PF GuC MMIO relay service, including handshake, runtime query, and GGTT update handling.
- Implements CTB `UPDATE_GGTT32` (`0x0102`) for the experimental Media 13 path.
- Uses Xe provisioning, workqueues, GPU scheduling, fences, and GGTT invalidation to complete VF updates.
- Performs Media 13 GGTT writes through privileged `MI_UPDATE_GTT` GPU batches, with VF region/owner validation.
- Adds work lifetime and generation checks around provisioning, FLR and CT reset. Concurrent reset behavior still needs hardware validation.

Verbose diagnostic logging is intentionally present in this initial research snapshot. It is not an upstream submission in its current form.

## Applying the patch

Download or clone this repository, then obtain the exact Intel base in a separate kernel checkout:

```sh
git init linux-xe-experiment
cd linux-xe-experiment
git remote add intel https://github.com/intel/mainline-tracking.git
git fetch --depth=1 intel c4227406af3b9ad8f8072b222ca06e0039f64441
git switch --detach FETCH_HEAD

# Replace /path/to with the location of this patch repository.
git apply --check /path/to/xe-sriov-mtl-arl/patches/linux-7.2.7-xe-mtl-arl-sriov-experimental.patch
git apply /path/to/xe-sriov-mtl-arl/patches/linux-7.2.7-xe-mtl-arl-sriov-experimental.patch
```

Verify the patch checksum using `sha256sum -c SHA256SUMS` from this repository. The [source manifest](verification/patched-source-SHA256SUMS) identifies the 20 resulting files. After applying, it can be checked from the kernel root with:

```sh
sha256sum -c /path/to/xe-sriov-mtl-arl/verification/patched-source-SHA256SUMS
```

Use a known working kernel configuration with Xe, PCI SR-IOV and Intel IOMMU support. Build and install through your normal kernel packaging procedure. Secure Boot systems need their own trusted kernel/module signing process.

## Tested boot environment

The host test used Arrow Lake-P `8086:7d51`, GuC **70.53.0** on both GTs, and these relevant parameters:

```text
intel_iommu=on iommu=pt xe.force_probe=7d51 i915.force_probe=!7d51 module_blacklist=i915 xe.max_vfs=3
```

`xe.max_vfs=3` sets the maximum; VFs must also be created through the PF's `sriov_numvfs` or your existing management service. The tested PF was `0000:00:02.0`; one of three created VFs was passed to Windows. Device IDs and PCI addresses depend on the machine.

Keep a known working kernel available and use an explicitly selected test boot. The tested fallback was an i915 6.17 kernel. [Test results and unresolved problems](docs/TESTING.md).

## AI-assisted development

The implementation was developed with **OpenAI Codex / LLM assistance**, through analysis of older i915 SR-IOV code, the official Intel i915 DKMS patch series, Xe paths for other enabled platforms, and public Intel technical manuals. The Xe integration follows its existing scheduling and resource model. [Methods, sources and attribution](docs/PROVENANCE.md).

## License

The touched Xe files and new ABI headers use **MIT SPDX identifiers**. Project additions are provided under the [MIT license](LICENSE); existing upstream notices remain applicable. The complete Linux kernel is governed by its own `COPYING` and per-file licenses. Public Intel manuals are referenced by links, with their own rights and terms.
