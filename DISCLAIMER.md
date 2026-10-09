# Experimental software disclaimer

This is an independent, experimental research project. It is not an official Intel release, is not endorsed or certified by Intel, and has not been accepted into upstream Linux or Intel mainline-tracking.

The patch forces a platform capability that the examined Intel Xe source does not enable for MTL/ARL. It adds VF/PF services based on public code and documentation. Enabling the flag does not establish complete hardware or driver support.

The latest RCC test used one Arrow Lake-P system, PCI device **8086:7d51**, and one Windows VF (VF2). Earlier bring-up used VF1. Meteor Lake hardware, other Arrow Lake devices, simultaneous guests with this corrected snapshot, concurrent resets, suspend/resume and long-term stability have not been validated.

The October 9 PF register permission corrects the reproduced A8/BGRA corruption and the measured Settings text defect. These results cover a specific ARL/VF2 configuration. All Windows UI elements, other graphics workloads and the complete SR-IOV feature set have not been certified. See [measured results and limits](docs/RCC_FIX.md).

Use this software only on systems reserved for experimentation. GPU hangs, host or guest instability, display loss, and loss of unsaved work are possible. Keep a working fallback kernel and a recovery path available.

The software is provided **AS IS**, without warranty, under the applicable licenses. This disclaimer does not modify the license terms or upstream copyright notices.

The code and documentation were developed with AI assistance. Review the implementation and the reported test limits before relying on it. Human review and additional testing are still required for any upstream proposal or broader support claim.
