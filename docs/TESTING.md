# Test status — 2026-10-09

## Hardware and software

| Component | Tested configuration |
| --- | --- |
| Host CPU / GPU | Intel Core Ultra 7 265H, Arrow Lake-P graphics `8086:7d51` |
| Kernel | Intel 7.2.7 base `c4227406af3b9ad8f8072b222ca06e0039f64441`, release `7.2.7-adv-xe-runtime5+` with the signed RCC Xe module |
| GuC | 70.53.0 on primary and media GT |
| Secure Boot | Enabled; kernel and modules signed with the tester's existing trusted key |
| VFs | Three created; latest RCC test uses VF2 in one guest; initial bring-up used VF1 |
| Guest | Windows 11 Pro |
| Guest display driver | Intel Arc 140T, `32.0.101.8132` |
| Capture | Looking Glass B7 / D12, kvmfr 0.0.12 |

## Current RCC result, October 9

The new render-engine permission for `COMMON_SLICE_CHICKEN3` (`0x7304`) corrects the reproduced A8/BGRA copy corruption and measured Settings text. The Windows driver remains 32.0.101.8132, with identical UMD/KMD file hashes.

- 24/24 native D3D11 cases correct across the three logical Intel adapters exposed by the same guest/VF.
- A8 remains `7f`, BGRA equals `ff332211`, and device-removed reasons remain zero.
- No command mutation, corrective RT flush or wait between the two copies. All 12 no-clear final PIPE_CONTROL values remain `00100001`.
- 8/8 real Settings captures have correct profile text, including after movement stops.
- During the probe, `0x7304` is permitted in all 355 qualified VF2 samples; bit 13 is active in 311.
- The loaded candidate module's build-id was checked against the signed module; all 552 reviewed Xe source hashes match the full patch's result on a clean Intel base.

The successful full snapshot retains diagnostic options and earlier experiments. The new functional difference from the immediately preceding failing build is the register permission. [Detailed method and limitations](RCC_FIX.md), [measured summary](../verification/rcc-20261009.json).

## Initial CTB progression, October 5

1. Enabling `.has_sriov` created VFs, but the original Xe implementation did not handle the observed GuC MMIO relay requests. The guest could not complete graphics initialization.
2. The MMIO service completed the initial GGTT bootstrap and the guest became reachable. Windows then sent CTB `UPDATE_GGTT32` (`0x0102`), which Xe rejected with `-EOPNOTSUPP`. DWM and Looking Glass waited for GPU synchronization.
3. The current MMIO + CTB patch completes those updates. D3D11 device creation and D3D12 COPY/DIRECT queue creation returned successfully. Looking Glass started capture and delivered changing 1920×1080 images.
4. Subsequent interactive testing revealed missing text/labels that reappear on mouse hover. A direct Windows `CopyFromScreen` capture also showed the defect. The October 9 RCC permission resolves the reproducer and Settings observations described above.

## Initial CTB recorded results

During the initial successful CTB run, approximately three minutes of guest activity were captured:

- 3,499 MMIO requests and 3,499 successful replies; no service/send errors.
- 954 completed CTB updates, covering 11,168 PTE writes, including repeated writes.
- Every recorded CTB result matched the expected PTE count. These hardware requests all used mode 0 with no extra copies.
- No relay errors, Xe GPU hang/reset, or TLB timeout in that captured interval.
- Intel D3D11 device creation: `S_OK`, feature level 11.0.
- D3D12 COPY and DIRECT: queue creation, empty command-list submission and fence completion succeeded. These empty-list probes do not test shader execution.
- Two Looking Glass frames, serials 6,479 and 9,071, contained visible desktop/WebGL content; 95.73% of pixels changed between them. The displayed WebGL FPS counter was not used as a benchmark, and Chrome's selected renderer was not independently identified.

The resulting 20 source files were checked against the compiled-source SHA-256 manifest. The kernel build completed. A C harness using ASan/UBSan passed 1,328,843 assertions covering GGTT bounds, owner handling, PAT, FIRST/LAST copy modes, expansion, chunking and 16 previously rejected real CTB payloads. Scheduler/fence operations in the harness were mocked; it does not validate concurrent hardware reset behavior.

## Open issues and untested cases

- The reproduced A8/BGRA corruption and measured Settings profile text are corrected. All Start elements, notification styles and other Windows graphics workloads have not been independently measured with this snapshot.
- Only one ARL-P device and one guest VF tested; no MTL hardware result.
- Simultaneous guests with the corrected snapshot, sustained load, concurrent FLR/reset and suspend/resume require validation.
- RDP was not separately retested after the RCC correction.
- GuC 70.53.0 is below the separate VF migration feature's 70.54.0 requirement; live migration was not tested.
- Privileged GPU GGTT writes and TLB synchronization are experimental and need broader review.

The reported observations are results from a specific test configuration, not a general support statement.
