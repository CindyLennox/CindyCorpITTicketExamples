## Issue

Archbook intermittently experiences a complete system/display freeze after YouTube has been playing in Firefox for an extended period.

When the freeze occurs:

- Mouse input stops responding.
- Keyboard shortcuts such as Alt+Tab do not respond.
- KDE Connect can no longer reach the system.
- The graphical session cannot be recovered normally.
- A forced shutdown using the power button is required.

## Environment

- System: Dell Inspiron 7375
- OS: Arch Linux
- Kernel: 7.2.4-arch1-2
- Desktop: KDE Plasma
- Display Protocol: Wayland
- GPU: AMD Radeon Vega 8 / Raven Ridge
- GPU Driver: amdgpu
- Mesa: 26.2.2
- Browser: Firefox 155.0.1
- Wi-Fi Adapter: Dell Wireless DW1810 / Qualcomm Atheros QCA9377
- Wi-Fi Driver: ath10k_pci

## Initial Hypothesis

Firefox GPU-accelerated rendering may be triggering an AMDGPU/display-stack hang during prolonged YouTube playback.

Logs from a previous failed session showed AMDGPU graphics and SDMA fence timeouts immediately before DRM display flip timeouts:

- `Fence fallback timer expired on ring gfx`
- `Fence fallback timer expired on ring sdma0`
- `flip_done timed out`

The Qualcomm Atheros QCA9377 Wi-Fi adapter was also observed generating numerous PCIe AER errors. These include:

- `RxErr`
- `BadDLLP`
- `BadTLP`
- `Timeout`

The Wi-Fi/PCIe problem may be independent of the graphics freeze or may contribute to overall system instability. At this stage, neither failure mechanism has been confirmed as the cause of the hard freeze.

## Test 1 — Firefox Hardware Acceleration

Firefox hardware-accelerated compositing was disabled using:

`layers.acceleration.disabled = true`

After restarting Firefox, `about:support` reported:

`Compositing: WebRender (Software)`

YouTube was then left playing continuously in Firefox to determine whether the hard freeze could be reproduced without GPU-accelerated browser compositing.

After approximately two hours of continuous playback using Software WebRender, no hard freeze had occurred.

This was encouraging but insufficient to establish Firefox hardware acceleration as the cause because the previous time-to-failure was not consistently established.

## Thermal Observations

System temperatures were checked during YouTube playback using `sensors`.

Observed temperatures:

- CPU (`Tctl`): ~76°C
- AMD GPU: ~76°C
- SODIMM: ~47°C
- CPU fan: ~5680 RPM / 6301 RPM maximum

The laptop was initially sitting on a mousepad and was subsequently moved to a hard surface to improve airflow.

This represents an environmental change during testing and must therefore be considered when interpreting stability results.

## Wi-Fi / PCIe Investigation

Inspection of the previous boot's kernel log revealed a sustained stream of PCIe AER Correctable errors associated with:

`0000:01:00.0`

The device was identified as:

`Qualcomm Atheros QCA9377 802.11ac Wireless Network Adapter [168c:0042]`

Subsystem:

`Dell Wireless DW1810 [1028:1810]`

Kernel driver:

`ath10k_pci`

The errors occurred repeatedly and included Physical Layer and Data Link Layer errors, `RxErr`, `BadDLLP`, and `Timeout`.

The PCIe errors were reproduced after reboot during a fresh session. Additional live monitoring produced another Data Link Layer error involving the QCA9377, including `BadTLP`.

This confirms that the QCA9377 PCIe/AER problem is reproducible and is not limited to the session that experienced the hard freeze.

However, the AER errors are reported as Correctable, so their presence alone does not establish that the QCA9377 caused the complete system freeze.

## Test 2 — PCIe ASPM

The current PCIe ASPM policy initially reported:

`[default] performance powersave powersupersave`

Inspection of the QCA9377 PCIe link showed:

`LnkCtl: ASPM L1 Enabled`

The ASPM policy was temporarily changed to:

`performance`

After the change, the QCA9377 PCIe link reported:

`LnkCtl: ASPM Disabled`

This establishes a second experimental configuration in which PCIe ASPM L1 is disabled for the QCA9377.

The change is temporary and will not be treated as a permanent fix until additional testing is completed.

## Current Test Configuration

The current extended-playback test now contains multiple changed variables compared with the original failure condition:

1. Firefox is using Software WebRender rather than hardware-accelerated compositing.
2. PCIe ASPM has been disabled on the QCA9377 through the `performance` ASPM policy.
3. The laptop is resting on a hard surface rather than a mousepad, potentially improving cooling.

Therefore, continued stability under the current configuration cannot be attributed to any single change.

No additional graphics, kernel, driver, hardware, or power-management changes should be made during the current test.

## Planned Isolation Testing

Future testing should change one variable at a time.

A useful control test is to return the laptop to its normal/default configuration and use it as an ordinary portable computer without the external-TV media setup.

Normal Wi-Fi activity, web browsing, and video playback can then be tested to determine whether the QCA9377 PCIe errors or complete system freeze occur during ordinary laptop use.

If necessary, the internal QCA9377 can also be disabled and networking provided through an external Wi-Fi adapter. Reproducing the same workload without the internal QCA9377 active would provide another method of isolating the internal Wi-Fi/PCIe path.

Kernel LTS testing and internal Wi-Fi hardware replacement remain possible later diagnostic steps, but neither is currently justified until the existing variables have been isolated.

## Additional Isolation Testing

Further testing demonstrated that disabling Firefox hardware acceleration did not eliminate the failure. The system subsequently experienced another hard freeze during prolonged/fullscreen YouTube playback while Firefox was using Software WebRender.

The internal Qualcomm Atheros QCA9377 was also investigated further. The adapter continued to generate reproducible PCIe AER Correctable errors, including `RxErr`, `BadDLLP`, `BadTLP`, and `Timeout`. Testing was also performed with the internal QCA9377 inactive/unbound and networking supplied externally.

Although the QCA9377 clearly exhibits a reproducible PCIe/AER problem, testing did not establish those correctable errors as the cause of the complete system freeze.

The display protocol was subsequently investigated. Plasma X11 initially survived prolonged playback, suggesting a possible Wayland-specific problem. However, the system later reproduced the hard freeze under X11 with HDMI output and prolonged video playback.

Because the failure could occur under both Wayland and X11, Wayland itself was no longer considered a sufficient explanation for the problem.

## Platform-Specific Power Management Investigation

Investigation of the Dell Inspiron 7375 and its AMD Raven Ridge platform identified a platform-specific Linux workaround involving the kernel parameter:

`idle=nomwait`

The Inspiron 7375 has documented Linux compatibility guidance involving:

`acpi_osi=Linux idle=nomwait`

The running system was verified to have `idle=nomwait` active before the final extended stability test.

## Final Stability Test

With `idle=nomwait` active, extended video playback was tested again under conditions representative of the previously problematic workload.

Testing included:

- KDE Plasma Wayland
    
- Firefox / YouTube playback
    
- Fullscreen video
    
- Internal laptop display
    
- Subsequent HDMI output to the external display
    

The Wayland + HDMI + fullscreen portion of the test began at approximately 07:43 on September 19, 2026.

After nearly 11 hours of operation, the system remained responsive and had not reproduced the hard freeze.

This represents substantially longer continuous operation than previous failing tests and includes the Wayland, HDMI, fullscreen-video workload that had previously been associated with the problem.

Continued subsequent use has also failed to reproduce the original hard freeze.

## Result

The original complete-system freeze is considered resolved.

The evidence does not support Firefox hardware acceleration as the root cause because the freeze was reproduced with Software WebRender.

The evidence also does not support Wayland alone as the root cause because the freeze was reproduced under Plasma X11.

The Qualcomm Atheros QCA9377 produces recurring PCIe AER Correctable errors, including `RxErr`, `BadDLLP`, `BadTLP`, and `Timeout`. These errors are being corrected by the PCIe error-recovery mechanism and have not produced an observable loss of network functionality or system performance. The system also remains stable despite their occurrence.

Accordingly, the QCA9377 AER messages are considered incidental to the hard-freeze issue rather than evidence of its root cause. No corrective action for the QCA9377 is currently required.

Extended stability testing after enabling the Raven Ridge idle-state workaround strongly indicates that the freezes were associated with platform CPU/power-management idle behavior.

## Resolution

Apply the kernel boot parameter:

`idle=nomwait`

Following application of this workaround, the Dell Inspiron 7375 successfully completed an extended Wayland + HDMI + fullscreen YouTube playback test lasting nearly 11 hours without reproducing the hard freeze.

Continued subsequent use has remained stable under workloads that previously reproduced the failure.

**Status: Resolved**

The recurring QCA9377 PCIe AER Correctable messages remain observable but have demonstrated no measurable operational impact and are not considered part of the resolved hard-freeze fault.