# Mac Won't Sleep

> **Applies to:** macOS 10.13 High Sierra — macOS 26.5 Tahoe · **Architecture:** Intel x86_64 & Apple Silicon ARM64

---

## Mac Won't Sleep — Power Management Assertion Holders, Network Wake Events, and SMC Sleep State Failures

> **[Full power assertion and wake source diagnostic](https://error-number-472173.github.io/.github/mac-wont-sleep)** — macOS 12 Monterey through macOS 26.5 Tahoe, Intel and Apple Silicon.

**What's actually responsible for Mac Won't Sleep, in order of architectural impact:**

- Hardware communication failure at the `IOKit` driver layer — detectable through Apple Diagnostics and `ioreg` independently of the user-visible symptom
- SMC state corruption affecting power, thermal, or hardware initialization sequences relevant to the affected subsystem
- Software configuration stored in property list (`.plist`) files that has become inconsistent across a macOS version transition
- Third-party kernel extensions (`kext`) or System Extensions interfering with the macOS subsystem responsible for Mac Won't Sleep
- `launchd`-managed daemon processes in a failed or restart-loop state due to dependency failures at boot
- Privacy permission entries in the TCC (Transparency, Consent, and Control) database blocking hardware access at the framework layer

---

## Internal macOS Architecture for Mac Won't Sleep

macOS organizes all system functionality into discrete architectural layers. Mac Won't Sleep involves the following components:

| Layer | Component | Relevance to Mac Won't Sleep |
|---|---|---|
| Hardware | Logic board, sensors, connected peripherals | Physical signal source and power delivery |
| Firmware | SMC, EFI, peripheral firmware | Low-level initialization and power state management |
| Kernel | XNU, IOKit, kexts / System Extensions | Hardware abstraction and driver communication |
| System daemons | `launchd`-managed services | Background process lifecycle management |
| Privacy framework | TCC database, `tccd` daemon | Per-app hardware access authorization |
| Configuration | `/Library/Preferences/`, `~/Library/Preferences/` | Stored settings and device state records |
| User interface | System Settings pane | User-facing configuration surface |

A failure at any of these layers produces the visible symptom of Mac Won't Sleep. The layer where the failure occurs determines the appropriate diagnostic approach — hardware-layer failures require IOKit inspection or Apple Diagnostics, while configuration-layer failures require plist analysis and daemon log review.

---

## APFS Volume Structure and Configuration File Architecture

Since macOS 10.13, all Mac startup volumes use **APFS (Apple File System)** organized into a container with multiple volumes sharing a unified free space pool:

| APFS Volume | Contents | Relevance to Mac Won't Sleep |
|---|---|---|
| Macintosh HD (System) | Sealed, read-only system files and frameworks | Cryptographically verified at each boot by SSV hash tree |
| Macintosh HD — Data | User data, apps, preferences, caches, logs | Writable; source of most configuration-related failures |
| VM | Swap files, `sleepimage` | Memory pressure and paging performance |
| Preboot | Boot metadata, firmware staging | Boot process and recovery orchestration |
| Recovery | recoveryOS, Disk Utility | Repair and reinstall environment |

The **Sealed System Volume (SSV)** introduced in macOS 11 Big Sur adds a Merkle hash tree over every file in the system volume. Corruption of any system file is detected at the next boot and remediated automatically before the login screen. Configuration failures in the writable Data volume are not covered by SSV and persist until explicitly addressed.

---

## SMC Architecture and Its Role in Mac Won't Sleep

The **System Management Controller (SMC)** is a dedicated microcontroller on the Mac logic board that operates below the operating system level. It manages thermal sensors, fan speed curves, power delivery sequencing, battery charging state machines, sleep and wake transitions, and hardware enable/disable signals for components including cameras, keyboards, and wireless modules.

The SMC maintains its own non-volatile state that persists across restarts and macOS updates. This state can become inconsistent after a kernel panic, unexpected power loss, or firmware update failure, producing hardware behavior anomalies that appear as software problems because they manifest at the driver level rather than causing visible hardware errors.

| SMC Domain | Components Managed | Failure Impact on Mac Won't Sleep |
|---|---|---|
| Thermal management | CPU/GPU throttling, fan curves | Unexpected throttling, fan behavior anomalies |
| Power sequencing | Component enable rails | Peripheral not recognized at boot |
| Battery management | Charge state machine, health tracking | Charging failures, inaccurate capacity readings |
| Sleep/wake | Sleep assertions, wake event routing | Sleep prevention, spurious wake events |
| Input hardware | Keyboard, trackpad, Touch Bar enable | Input device not recognized |

---

## TCC Privacy Database and Hardware Access Architecture

Since macOS 10.14 Mojave, access to hardware inputs is governed by the **TCC (Transparency, Consent, and Control)** subsystem. The `tccd` daemon manages two SQLite databases mapping application bundle identifiers to hardware access decisions:

| Database | Path | Scope |
|---|---|---|
| System TCC | `/Library/Application Support/com.apple.TCC/TCC.db` | System-wide hardware access grants |
| User TCC | `~/Library/Application Support/com.apple.TCC/TCC.db` | Per-user app authorization records |

A corrupted or inconsistent TCC database produces hardware access failures that appear at the framework level — the hardware is fully functional, but the framework layer refuses to pass data to the requesting application. TCC corruption is a common cause of Mac Won't Sleep appearing after a macOS update, because the update may invalidate stored bundle identifier checksums without regenerating the access grants.

---

## Diagnostic Data Sources

| Tool | Access | What It Reveals for Mac Won't Sleep |
|---|---|---|
| Console.app | Applications → Utilities | Daemon crash logs, restart loops, TCC denial events |
| Activity Monitor | Applications → Utilities | CPU and memory usage per process, real-time resource pressure |
| System Information | About This Mac → System Report | Hardware inventory, IOKit device tree, power and battery history |
| Disk Utility First Aid | Applications → Utilities or Recovery | APFS container and volume integrity errors |
| Apple Diagnostics | Hold **D** at boot | Hardware-level component fault codes independent of macOS |
| `ioreg -l` | Terminal | Full IOKit registry dump including device properties and driver binding state |
| `pmset -g log` | Terminal | Power management event log — sleep, wake, assertions, and charge events |
| `log` CLI | Terminal | Structured Unified Log query with subsystem and process filtering |

---

## Safe Boot Diagnostic Environment

Safe Boot starts macOS with all third-party kernel extensions and System Extensions disabled, most LaunchAgents suppressed, and `fsck_apfs` run automatically on the startup volume:

- **Intel Mac:** Hold **Shift** immediately after pressing the power button
- **Apple Silicon Mac:** Hold **Power** until "Loading startup options" appears, then hold **Shift** while clicking Continue

If Mac Won't Sleep does not occur in Safe Boot, the cause is categorically in third-party software or accumulated user configuration loaded during normal boot — not in hardware or core macOS components.

---

## New User Account Isolation

Creating a temporary standard user account (System Settings → Users & Groups → Add Account) and reproducing the conditions that trigger Mac Won't Sleep:

- **Problem absent in new account** → cause is in the original user's `~/Library/` directory: preferences, LaunchAgents, Application Support data, TCC entries, or login items
- **Problem present in new account** → cause is system-wide: a system daemon, `/Library/` configuration, hardware, or macOS itself

---

## macOS Version Architecture Context

| macOS Version | Darwin Kernel | Relevant Architectural Change |
|---|---|---|
| macOS 12 Monterey | Darwin 21 | System Extensions replace most third-party kexts |
| macOS 13 Ventura | Darwin 22 | Revised System Settings; some plist paths changed |
| macOS 14 Sonoma | Darwin 23 | Stricter background process and hardware access controls |
| macOS 15 Sequoia | Darwin 24 | Revised daemon lifecycle for several subsystems |
| macOS 26 Tahoe | Darwin 25 | Updated sandboxing and extension entitlement model |

---

## Hardware vs Software Decision Matrix

| Evidence | Diagnostic Conclusion |
|---|---|
| Problem absent in Safe Boot | Third-party extension or login item is the cause |
| Problem absent in new user account | User-level `~/Library/` configuration is the cause |
| TCC denial events in Console.app | Privacy permission database inconsistency |
| Problem present in Recovery environment | Hardware or firmware cause |
| Apple Diagnostics reports fault code | Hardware failure confirmed |
| Problem began immediately after macOS update | Update compatibility issue; check for vendor updates |
| SMC reset changes behavior | SMC state corruption was contributing to the issue |

---

*This reference covers Mac Won't Sleep as observed on macOS 10.13 High Sierra through macOS 26.5 Tahoe on Intel x86_64 and Apple Silicon ARM64 hardware.*
