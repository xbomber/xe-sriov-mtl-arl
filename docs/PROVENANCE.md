# Development provenance and references

## AI assistance

This patch was developed with **OpenAI Codex and LLM assistance** during an interactive investigation directed and tested by Marco Mondin.

The assistance included source comparison, protocol interpretation, implementation, build orchestration, software harnesses, hardware/guest diagnostics, and documentation. The work compared:

1. Older i915 SR-IOV implementation in the working 6.17 bbaa-bbaa branch.
2. The official Intel i915 DKMS/quilt SR-IOV implementation on Linux 6.18.19.
3. Xe source for other platforms whose descriptors already enable SR-IOV, including Alder Lake-P and newer platform paths.
4. Public Intel technical manuals and source-defined GuC ABI headers.

Public documentation supplied architectural context. The i915 source and its ABI definitions supplied the concrete VF/PF protocol and Media 13 binder reference. The new implementation uses Xe scheduling, provisioning, workqueue, fence and invalidation mechanisms.

The manuals consulted are not a complete public MTL/ARL firmware or errata specification. Platform similarities are investigative references and do not prove that the MTL/ARL implementation is complete.

## Source references

- [Intel mainline-tracking base commit](https://github.com/intel/mainline-tracking/commit/c4227406af3b9ad8f8072b222ca06e0039f64441), Linux 7.2.7.
- [Working i915 6.17 bbaa-bbaa source](https://github.com/bbaa-bbaa/intel-mainline-tracking/tree/2d4f222bcdd23329a7c61af22f89fec41d9ab25d).
- [Official Intel edge-gfx DKMS installer](https://github.com/intel/edge-gfx-dkms-installer).
- [Intel Linux quilt 6.18 series](https://github.com/intel/linux-intel-quilt/tree/6.18/linux). Reconstructed on Linux 6.18.19 for comparison with vanilla i915.
- [Intel Graphics SR-IOV Toolkit](https://github.com/intel/GFX-SRIOV-Toolkit).

Relevant i915 components examined include `intel_iov_service.c`, `intel_iov_ggtt.c`, `intel_ggtt.c`, the GuC MMIO/CTB relay ABI headers, and MTL-specific runtime register handling. Relevant Xe components include platform descriptors, PF service/relay handling, GGTT provisioning, migration queues, ring submission and TLB invalidation.

## Render color cache permission, October 9

The latest fix follows a Windows driver request to set bit 13 of `COMMON_SLICE_CHICKEN3`, a missing permission observed in VF-qualified PF register samples, and a reproducible D3D11 A8-to-BGRA copy failure. The PF permission change restores correct copies and the measured Settings text without changing the Windows driver. [Investigation, results and limitations](RCC_FIX.md).

Relevant primary source references:

- [i915 register whitelist commit cebc13de](https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux/+/cebc13de7e704b1355bea208a9f9cdb042c74588).
- [Xe per-queue programming of bit 13, commit 0e07b163](https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux/+/0e07b16371b6eef9b5a4a1fd3e7942938811072e).
- [Mesa BTP+BTI RCC keying, commit af2d51ea](https://chromium.googlesource.com/external/gitlab.freedesktop.org/mesa/mesa/+/af2d51eafa9969c5240b6ac4b8902e787513f3a6).
- [Iris binding-table state and render-target flush requirements](https://chromium.googlesource.com/external/gitlab.freedesktop.org/mesa/mesa/+/87afa3193eee4c6dbdef4306a7c6ead6a5a4d825/src/gallium/drivers/iris/iris_state.c).

These references explain the hardware mode. They do not establish a Vulkan path for the affected Windows UI.

## Public Intel technical documents

- [Intel Core Ultra Network and Edge Platforms datasheet addendum, document 793432](https://cdrdv2-public.intel.com/793432/793432_Intel_Core_Ultra_Datasheet_Rev001.pdf), February 2024, pages 13–16: graphics SR-IOV, GuC and VF/PF communication.
- [Intel BXT PRM, Volume 5: Command Stream Programming, document 685502](https://cdrdv2-public.intel.com/685502/intel-gfx-prm-osrc-bxt-vol05-command-stream-programming.pdf): historical reference for privileged commands and GGTT/PPGTT context. It is not a complete MTL/ARL specification.
- [Intel public graphics programmer reference catalog](https://www.intel.com/content/www/us/en/docs/graphics-for-linux/developer-reference/1-0/overview.html).
- [Intel GuC KMD API overview](https://www.intel.com/content/www/us/en/docs/graphics-for-linux/developer-reference/1-0/guc-kmd-api.html).
- [Intel Meteor Lake datasheet: SR-IOV](https://edc.intel.com/content/www/us/en/design/products/platforms/details/meteor-lake-u-p/core-ultra-processor-datasheet-volume-1-of-2/single-root-i-o-virtualization-sr-iov/).

Only public references were used. Restricted Intel RDC material was not obtained. Vendor documentation is linked here and is governed by its own terms.

## Extraction and verification

The initial October 5 patch contained two new ABI headers and 18 modified Xe files. The October 9 full patch is freshly extracted against the same Intel base from the exact source used for the loaded signed RCC module. It changes 33 Xe files. A separate checkout verifies application, reverse application and every one of the 552 reviewed Xe source hashes. The published manifest lists the 33 changed files.

The single-file RCC patch was also applied independently to the Intel base; its resulting whitelist matches the boot-tested source. That check establishes source application, while the hardware test used the cumulative full snapshot with its documented boot options. The full snapshot retains diagnostic instrumentation and earlier experiments so it can reproduce the tested source; the standalone diff identifies the new functional permission change.

The repository records an experimental source snapshot. Review by Intel/Linux maintainers has not taken place. Human review remains necessary before upstream submission, including review of code provenance and licensing.

Relevant guidance: [AI coding assistants](https://docs.kernel.org/process/coding-assistants.html), [tool-generated content](https://docs.kernel.org/process/generated-content.html), and [kernel licensing](https://docs.kernel.org/process/license-rules.html).
