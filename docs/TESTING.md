# Test status — 2026-10-05

## Hardware and software

| Component | Tested configuration |
| --- | --- |
| Host CPU / GPU | Intel Core Ultra 7 265H, Arrow Lake-P graphics `8086:7d51` |
| Kernel | Intel 7.2.7 base `c4227406af3b9ad8f8072b222ca06e0039f64441`, patched release `7.2.7-adv-xe-ggtt-ctb+` |
| GuC | 70.53.0 on primary and media GT |
| Secure Boot | Enabled; kernel and modules signed with the tester's existing trusted key |
| VFs | Three created; **only VF1 tested in a guest** |
| Guest | Windows 11 Pro |
| Guest display driver | Intel Arc 140T, `32.0.101.8132` |
| Capture | Looking Glass B7 / D12, kvmfr 0.0.12 |

## Progression

1. Enabling `.has_sriov` created VFs, but the original Xe implementation did not handle the observed GuC MMIO relay requests. The guest could not complete graphics initialization.
2. The MMIO service completed the initial GGTT bootstrap and the guest became reachable. Windows then sent CTB `UPDATE_GGTT32` (`0x0102`), which Xe rejected with `-EOPNOTSUPP`. DWM and Looking Glass waited for GPU synchronization.
3. The current MMIO + CTB patch completes those updates. D3D11 device creation and D3D12 COPY/DIRECT queue creation returned successfully. Looking Glass started capture and delivered changing 1920×1080 images.
4. Subsequent interactive testing revealed missing text/labels that reappear on mouse hover. A direct Windows `CopyFromScreen` capture also shows the defect. **Rendering remains incomplete.**

## Recorded results

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

- Missing fonts/application labels; redraw on hover changes visibility. The cause is still under investigation.
- Only one ARL-P device and one guest VF tested; no MTL hardware result.
- Multiple active guests, sustained load, concurrent FLR/reset and suspend/resume require validation.
- RDP was not separately retested after the CTB correction.
- GuC 70.53.0 is below the separate VF migration feature's 70.54.0 requirement; live migration was not tested.
- Privileged GPU GGTT writes and TLB synchronization are experimental and need broader review.

The reported observations are results from a specific test configuration, not a general support statement.
