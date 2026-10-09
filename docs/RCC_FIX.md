# Windows VF render color cache permission — 2026-10-09

The new functional change permits read/write access to `COMMON_SLICE_CHICKEN3` (`0x7304`) on the render engine of MTL/ARL SR-IOV PFs, Graphics 12.70–12.74. The Windows UMD can then request bit 13, `DISABLE_STATE_CACHE_PERF_FIX`, which selects BTP+BTI render color cache keying.

The PF patch grants permission through Xe's existing register whitelist. It does not force the bit or inject a render-target flush into Windows batches. [Standalone functional diff](../patches/linux-7.2.7-xe-mtl-arl-rcc-permission.patch).

## Evidence before the change

The Windows Intel UMD was observed calling its register-write helper 80 times with address `0x7304` and masked value `0x20002000`. Static inspection of that helper identifies the corresponding LRI emission. The call-site trace establishes the request; it does not prove hardware acceptance.

In 321 VF2-qualified PF samples, bit 13 was zero and the render whitelist lacked `0x7304`.

A minimal D3D11 sequence reproduces corruption:

1. Create an A8 32×32 atlas and immutable 1×1 source containing `7f`; copy the source texel to the atlas.
2. Create a BGRA8 1024×832 atlas and immutable 1×1 source containing `ff332211`; copy the source texel to that atlas.
3. Keep both copies in the original sequence, with no query, Flush, Map or completion wait between them.
4. Complete GPU work, read the BGRA result through staging, then read the A8 atlas and source.

All four variants without a BGRA clear failed: same-device readback, shared producer, shared consumer and shared/GDI consumer. BGRA was zero; the later A8 readback was `33` or `ff`. The source remained correct. All four variants with a clear before BGRA copy passed.

Adding only `PIPE_CONTROL_RENDER_TARGET_CACHE_FLUSH` (`0x1000`) to the final command corrected the copies. A sham write of the original DWORD retained the failure. A/B/A repeated 4 failures, 0 failures, 4 failures. A guarded one-bit change on the real Settings startup copy similarly produced white/black/white profile text in a B/A/B test.

State-cache and texture-cache invalidations did not correct the reproducer. Separating the copies with D3D11 Flush or a completion wait did, but those variants change submission/barrier behavior and used white rather than tagged source values. They are not a pure latency experiment.

## Results after the PF permission change

The signed candidate module was verified through the build-id note of the loaded Xe module. The host kernel image, firmware and Windows driver were unchanged.

| Measure | Result |
| --- | --- |
| VF2 samples during the native probe | 355 qualified brackets |
| `0x7304` permission | Present in 355/355 |
| Bit 13 | Observed active in 311/355 |
| D3D11 copy/readback | 24/24 cases correct |
| Tagged A8 atlas | `7f` in all 24 cases |
| BGRA atlas and source | `ff332211` in all 24 cases |
| Original final PIPE_CONTROL in the 12 no-clear cases | `00100001`, RT flush bit still absent |
| Real Settings profile text | Correct in 8/8 captures, including stationary frames |

The guest exposed three logical Intel 8086:7d51 DXGI adapters. Each was selected explicitly and passed the same eight cases; these are not three physical GPUs or three guest VFs. Software adapters/WARP were excluded.

Reboot changed DXGI LUIDs. The probe was recompiled to enumerate/select the new identities and report actual adapter flags. Every GPU test helper and the entire copy/completion/readback function were unchanged; the postboot EXE is not byte-identical to the preboot EXE. No debugger or command-buffer mutation was used in the passing run. The original final PIPE_CONTROL was also checked from captured CPU command bytes.

Settings was a fresh process in the new guest boot. Forty automated resizes and navigation actions produced eight correct real desktop captures. The measured profile regions contained 457 and 701 bright pixels in each frame; the earlier uncorrected control had zero in both. This measures framebuffer brightness, not texture alpha. Window placement and cursor position were restored.

[Machine-readable measured summary](../verification/rcc-20261009.json).

## Interpretation

The results identify the missing PF permission as an effective correction for this reproduced ARL/VF2 defect. Windows requests the cache-keying mode, that mode can now be observed active on the VF, and the copies/UI work without adding the previously necessary flush.

The explanation is consistent with the BTP+BTI cache-keying contract and BTI reuse requirements described in the [primary source references](PROVENANCE.md#render-color-cache-permission-october-9). It does not imply that Windows routes this UI through Vulkan.

## Scope and limits

- The hardware test used the complete cumulative source snapshot and `xe.mtl_sriov_full_restore=1 xe.mtl_sriov_gpu_assign=1`. These options and earlier Blend Fill tuning predate the permission change and remain in the full patch.
- The standalone diff reproduces the tested whitelist on the original Intel base. The original October 5 snapshot plus only that diff has not been separately boot-tested.
- PF samples compare LRCA/status/GGTT PTE ownership before and after, with stable observed MCR steering. These are sequential observations without a context pause or MCR lock; they cannot exclude ABA or identify a particular draw atomically.
- Some qualified samples have bit 13 zero: the guest chooses the mode per context/workload. The probe/UI passed; no forced global bit value was added.
- The copy reproducer does not implement the original DirectComposition GUARDED resources or NT-handle sharing.
- This test did not repeat a return to the old PF module for a boot-level A/B/A, simultaneous guests, reset/suspend, all MCR replicas, physical MTL hardware or every Windows UI element.

The target defect is corrected in this configuration. Complete SR-IOV support and long-term stability remain experimental.
