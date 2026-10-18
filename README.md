# UEFI TCG Measured Boot Replay & Attestation Reconstruction

Bare-metal UEFI architecture demonstrating TPM 2.0 Measured Boot state reconstruction and synthetic TCG2 protocol event log replay under active, hardware-rooted Secure Boot.

## Overview

Hardware-rooted Measured Boot architectures rely on the foundational assumption that values held within the Trusted Platform Module (TPM 2.0) Platform Configuration Registers (PCRs) are exclusively extended by authentic platform firmware during early boot initialization. This trust model underpins remote attestation verifiers, BitLocker sealed key release (PCR7 binding), and endpoint security integrity checks (EDR / Anti-Cheat).

However, this measurement guarantee is bounded by platform firmware configuration. In mainstream UEFI implementations (notably American Megatrends AMI Aptio, InsydeH2O, and Phoenix SCT), the active TCG specification standard is controlled via a persistent NVRAM byte within the platform `Setup` variable. When toggled from `TCG 2.0` to `TCG 1.2`:
1. The UEFI PEI/DXE firmware skips TPM 2.0 measured boot initialization routines.
2. The discrete or firmware TPM 2.0 device remains fully powered and accessible over its Command Response Buffer (CRB) memory-mapped I/O (MMIO) interface.
3. The TPM 2.0 SHA-256 PCR banks remain in their unextended, zero-initialized reset state (`0x00000000...`).

By orchestrating a two-stage pre-OS boot sequence via a Microsoft-signed first-stage shim (`bootmgfw.efi` proxy) and an enrolled Machine Owner Key (MOK), unsigned pre-OS code executes while `SecureBoot` remains genuinely asserted. The custom loader interfaces with the TPM 2.0 CRB registers directly, sequentially replays a clean, pre-recorded reference TCG event log into the vacant SHA-256 PCR banks, registers a displacement `EFI_TCG2_PROTOCOL`, purges all MOK/SBAT transient NVRAM variables, restores the firmware setup variable to `TCG 2.0`, and transfers execution seamlessly to the authentic Windows Boot Manager (`bootmgr`).

```
                +-------------------------------------------------------+
                |           Firmware Initialization (SEC / PEI)         |
                |    Intel Boot Guard / AMD PSB Hardware Root of Trust  |
                +-------------------------------------------------------+
                                           |
                                           v
               +---------------------------------------------------------+
               |   Setup[0x1B] == TCG 1.2: Firmware skips TPM2 extends   |
               |   TPM 2.0 SHA-256 PCR banks 0-7 remain pristine zeros   |
               +---------------------------------------------------------+
                                           |
                                           v
               +---------------------------------------------------------+
               |        Microsoft-Signed First Stage Shim (db trusted)   |
               |        Validates MOK-signed ib-loader.efi (grubx64)     |
               +---------------------------------------------------------+
                                           |
                                           v
               +---------------------------------------------------------+
               |                     ib-loader.efi                       |
               |  1. ACPI Discovery -> CRB MMIO Base Address Mapping     |
               |  2. Replay genuine tcglog.ib into PCR0-PCR7 via CRB     |
               |  3. Install Synthetic EFI_TCG2_PROTOCOL                 |
               |  4. Delete MOK, SBAT, SSP Variables                     |
               |  5. Restore Setup[0x1B] = TCG 2.0                       |
               |  6. Execute Target Pre-OS Payload                       |
               |  7. Chain-load Restored Windows bootmgfw.efi            |
               +---------------------------------------------------------+
                                           |
                                           v
               +---------------------------------------------------------+
               |             Windows Boot Manager (bootmgfw.efi)         |
               |   Attestation: BitLocker PCR7 Valid, GetVariable Valid  |
               +---------------------------------------------------------+
```

## Technical Specifications

- **Target Architecture**: x86_64
- **Target Firmware**: UEFI Specification 2.7+ (EDK II compatible)
- **Host Deployment Systems**: Windows 10 / 11 (AMD64), Linux (x86_64 cross-compilation)
- **TPM Interface**: TPM 2.0 Command Response Buffer (CRB) MMIO Specification
- **Hardware Security Compliance**: Fully compliant with Intel Boot Guard (ACM) and AMD Platform Secure Boot (PSB)
- **Integrity Signal States**:
  - `GetVariable("SecureBoot")`: Returns `1` (`TRUE`) without hook interception or runtime patching.
  - `PCR[0-7]` (SHA-256): Cryptographically identical to authentic hardware boot chain.
  - `TCG Event Log`: Byte-for-byte replica of authentic baseline log with complete event sequence.
  - `BitLocker (PCR7)`: Integrity binding intact; sealed key release succeeds without recovery prompt.

## Memory & Register Layout

### 1. TPM 2.0 Command Response Buffer (CRB) MMIO Space

The CRB interface is discovered via the ACPI TPM2 table (`TPM2_ACPI_TABLE`) and mapped into standard MMIO physical address space (typically base address `0xFED40000`).

| Register Offset | Register Name | Width | Description |
| :--- | :--- | :--- | :--- |
| `0x0008` | `CRB_LOC_CTRL` | 32-bit | Locality Control: Request locality (`bit 0`), Relinquish locality (`bit 1`) |
| `0x000C` | `CRB_LOC_STS` | 32-bit | Locality Status: Locality granted (`bit 0`), Locality active state |
| `0x0030` | `CRB_INTF_ID` | 32-bit | Interface Identification: CRB interface descriptor (`bits 3:0 == 0x01`) |
| `0x0040` | `CRB_CTRL_REQ` | 32-bit | Control Request: `cmd_ready` (`bit 0`), `go_idle` (`bit 1`) |
| `0x0044` | `CRB_CTRL_STS` | 32-bit | Control Status: `tpm_idle` (`bit 0`), `tpm_sts` error indicator (`bit 1`) |
| `0x0048` | `CRB_CTRL_CANCEL` | 32-bit | Command Cancel: Abort current execution |
| `0x004C` | `CRB_CTRL_START` | 32-bit | Execution Start: Write `1` to begin hardware execution of command buffer |
| `0x0050` | `CRB_CTRL_CMD_SIZE` | 32-bit | Maximum physical command buffer allocation size |
| `0x0058` | `CRB_CTRL_CMD_LADDR` | 32-bit | Low 32 bits of physical command buffer address |
| `0x005C` | `CRB_CTRL_CMD_UADDR` | 32-bit | High 32 bits of physical command buffer address |
| `0x0060` | `CRB_CTRL_RSP_SIZE` | 32-bit | Maximum physical response buffer allocation size |
| `0x0068` | `CRB_CTRL_RSP_ADDR` | 64-bit | 64-bit physical response buffer address |

### 2. Firmware NVRAM Setup Variable Structure

The TCG specification selector is stored in the persistent non-volatile `Setup` variable:
- **Vendor GUID**: `EC87D643-EBA4-4BB5-A1E5-3F3E36B20DA9` (AMI Aptio Setup GUID)
- **Variable Name**: `L"Setup"`
- **Attributes**: `EFI_VARIABLE_NON_VOLATILE | EFI_VARIABLE_BOOTSERVICE_ACCESS | EFI_VARIABLE_RUNTIME_ACCESS` (`0x00000007`)
- **Default Offset**: `0x1B` (AMI Aptio V). Configurable per vendor implementation.

| Value | Specification Mode | Firmware Behavior |
| :--- | :--- | :--- |
| `0x00` | `TCG 1.2` | Firmware suppresses TPM 2.0 PEI/DXE measurements; SHA-256 PCR banks remain `0x00` |
| `0x01` | `TCG 2.0` | Standard TPM 2.0 Measured Boot; firmware measures SEC/PEI/DXE into PCRs |

### 3. TPM2 Command Wire Format: PCR Extend (`TPM2_CC_PCR_Extend`)

Commands dispatched over the CRB physical command buffer adhere to the TPM 2.0 specification frame layout:

```
+-------------------------------------------------------------------------+
| TPM Command Header                                                      |
|   tag: TPM_ST_SESSIONS (0x8002)                                [2 bytes] |
|   commandSize: total frame length                              [4 bytes] |
|   commandCode: TPM2_CC_PCR_Extend (0x00000182)                 [4 bytes] |
+-------------------------------------------------------------------------+
| Target PCR Handle                                                       |
|   pcrHandle: TPMI_DH_PCR (0x00000000 - 0x00000007)             [4 bytes] |
+-------------------------------------------------------------------------+
| Authorization Area                                                      |
|   authSize: Size of authorization sessions (9 bytes)           [4 bytes] |
|   sessionHandle: TPM_RS_PW (0x40000009)                        [4 bytes] |
|   nonce: Empty (0 bytes)                                       [2 bytes] |
|   sessionAttributes: 0x00                                      [1 byte]  |
|   auth: Empty password (0 bytes)                               [2 bytes] |
+-------------------------------------------------------------------------+
| Payload: Digested Values                                                |
|   count: 1                                                     [4 bytes] |
|   hashAlg: TPM_ALG_SHA256 (0x000B)                             [2 bytes] |
|   digest: 32-byte SHA-256 event digest                        [32 bytes] |
+-------------------------------------------------------------------------+
```

## Two-Stage Execution Lifecycle

```
 Stage 1: Baseline Capture (Host)
 +-----------------------------------------------------------------------+
 | 1. Host booted under standard TCG 2.0 configuration.                  |
 | 2. Execute `ib-tcg-dump.exe` with SeSystemEnvironmentPrivilege.       |
 | 3. Query TBS (TPM Base Services) to retrieve clean TCG 2.0 Event Log. |
 | 4. Serialize to disk as reference snapshot `tcglog.ib`.               |
 +-----------------------------------------------------------------------+
                                     |
                                     v
 Stage 2: Deployment & Provisioning (Host)
 +-----------------------------------------------------------------------+
 | 1. Generate RSA-2048 / SHA-256 Machine Owner Key (MOK) pair.          |
 | 2. Sign `ib-loader.efi` Authenticode structure with MOK certificate.  |
 | 3. Backup authentic `EFI\Microsoft\Boot\bootmgfw.efi`.                |
 | 4. Install Microsoft-signed `shimx64.efi` as primary `bootmgfw.efi`.  |
 | 5. Stage `ib-loader.efi` as `grubx64.efi`, MokManager as `mmx64.efi`. |
 | 6. Write `ib-config.bin` containing Setup variable offset (0x1B).     |
 +-----------------------------------------------------------------------+
                                     |
                                     v
 Stage 3: Boot 1 - Configuration Toggle (UEFI)
 +-----------------------------------------------------------------------+
 | 1. Firmware loads Microsoft-signed shim -> MokManager prompts user.   |
 | 2. User confirms physical-presence MOK enrollment (Password: 1234).   |
 | 3. Shim executes `ib-loader.efi` under verified MOK trust.            |
 | 4. Loader reads `Setup` variable; observes `TpmSpecVersion == 2.0`.   |
 | 5. Loader modifies `Setup[0x1B] = 0x00` (TCG 1.2 mode).               |
 | 6. Loader triggers warm system reset via `ResetSystem()`.             |
 +-----------------------------------------------------------------------+
                                     |
                                     v
 Stage 4: Boot 2 - Replay & Handoff (UEFI)
 +-----------------------------------------------------------------------+
 | 1. Firmware boots in TCG 1.2 mode; skips TPM 2.0 PCR extends.         |
 | 2. Shim executes `ib-loader.efi`. Setup indicates TCG 1.2 active.     |
 | 3. Restore authentic `bootmgfw.efi` to disk; delete staging binaries. |
 | 4. Initialize uACPI subsystem; locate TPM2 ACPI table and CRB MMIO.   |
 | 5. Parse `tcglog.ib`; issue sequential `TPM2_CC_PCR_Extend` commands  |
 |    directly to hardware CRB interface, populating PCRs 0 through 7.   |
 | 6. Install synthetic `EFI_TCG2_PROTOCOL`, redirecting GetEventLog()   |
 |    queries to genuine log buffer and sinking subsequent extends.      |
 | 7. Delete `MokList`, `MokListRT`, `SbatLevel`, `SSP` NVRAM variables. |
 | 8. Restore `Setup[0x1B] = 0x01` (TCG 2.0) for future boots.           |
 | 9. Load and execute target pre-OS payload in physical memory.         |
 | 10. Invoke `gBS->StartImage()` on restored Windows `bootmgfw.efi`.     |
 +-----------------------------------------------------------------------+
```

## Attestation State Comparison

The table below demonstrates the verifiable telemetry perceived by remote attestation authorities and anti-cheat driver verifiers:

| Integrity Telemetry | Standard Clean Boot | Standard Hooked Boot | uefi-tcg-measured-boot-replay |
| :--- | :--- | :--- | :--- |
| `SecureBoot` Variable | `0x01` (Active) | Patched in memory | `0x01` (Genuinely asserted in NVRAM) |
| Intel Boot Guard / AMD PSB | Passed (ACM verified) | Passed | Passed (Firmware unmodified) |
| PCR[0-7] SHA-256 Banks | Extended by Firmware | Mismatched / Random | Mathematically Identical to Clean Boot |
| TCG Event Log | Genuine Firmware Log | Missing / Corrupted | Genuine Replayed Baseline Log |
| BitLocker PCR7 Release | Key released | Trips Recovery Key | Key released cleanly without prompt |
| Remote Attestation (Azure/EAC) | Valid Token Issued | Attestation Rejected | Valid Token Issued |
| MOK / SBAT Trace Residue | None | Present in NVRAM | Completely Purged prior to Windows Handoff |

## Project Structure

```
.
├── .cargo/
│   └── config.toml               # Target specification (x86_64-unknown-uefi)
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── defect_report.md      # Bug and issue reporting template
│   │   └── research_inquiry.md   # Firmware research query template
│   ├── workflows/
│   │   └── build.yml             # GitHub Actions CI for UEFI and Host tools
│   └── pull_request_template.md  # PR contribution standard
├── assets/
│   └── comparison.png            # Visual verification of identical PCR bank hashes
├── crates/
│   ├── ib-config/                # ib-config.bin structure definitions and serialization
│   ├── ib-loader/                # Core UEFI application entry point and orchestration
│   ├── ib-tcg2/                  # Synthetic EFI_TCG2_PROTOCOL displacement implementation
│   ├── ib-tcglog/                # TCG Event Log parsing, verification, and serialization
│   ├── ib-tpm-crb/               # Bare-metal TPM 2.0 Command Response Buffer (CRB) driver
│   ├── ib-tpm2/                  # TPM 2.0 frame marshalling and PCR extend structures
│   ├── ib-uacpi/                 # Bare-metal ACPI table traversal and parsing wrapper
│   └── uacpi-sys/                # Raw C FFI bindings to uACPI framework
├── docs/
│   └── ARCHITECTURE.md           # In-depth architectural treatise on TCG2 and CRB MMIO
├── host/
│   ├── ib-install/               # Windows host deployment and MOK provisioning tool
│   ├── ib-tcg-dump/              # Windows host baseline TCG Event Log acquisition utility
│   ├── Cargo.lock                # Host workspace dependency resolution
│   └── Cargo.toml                # Host workspace manifest
├── third-party/
│   └── uacpi/                    # uACPI upstream submodule
├── .gitignore                    # Build and artifact exclusions
├── .gitmodules                   # Git submodule configuration
├── Cargo.toml                    # Root UEFI workspace manifest
├── CONTRIBUTING.md               # Guidelines for research contributions
├── LICENSE                       # MIT License
├── LICENSE-APACHE                # Apache License Version 2.0
├── LICENSE-MIT                   # MIT License
├── rust-toolchain.toml           # Toolchain pinning
├── rustfmt.toml                  # Code style definition
└── SECURITY.md                   # Security vulnerability reporting policy
```

## Build & Toolchain

### Prerequisites

1. **Rust Toolchain**: Nightly or 2024 edition with `x86_64-unknown-uefi` target installed:
   ```bash
   rustup target add x86_64-unknown-uefi
   ```
2. **C Toolchain**: LLVM Clang, `llvm-ar`, and `libclang` (required for uACPI bindings generation).
3. **Submodule Initialization**:
   ```bash
   git submodule update --init --recursive
   ```

### Building the UEFI Loader (`ib-loader.efi`)

Compile the standalone `no_std` UEFI application:
```bash
cargo build --release --target x86_64-unknown-uefi
```
The resulting binary is generated at:
`target/x86_64-unknown-uefi/release/ib-loader.efi`

### Building Host Utilities (`ib-install.exe` and `ib-tcg-dump.exe`)

Navigate to the `host/` workspace:
```bash
cd host
cargo build --release --target x86_64-pc-windows-msvc
```
Binaries will be output to `host/target/x86_64-pc-windows-msvc/release/`.

### Validation under QEMU / OVMF

Smoke-testing the loader under OVMF with a virtual TPM 2.0 CRB emulator:
```bash
mkdir -p esp/EFI/BOOT tpmstate
cp target/x86_64-unknown-uefi/release/ib-loader.efi esp/EFI/BOOT/BOOTX64.EFI
cp /usr/share/edk2/x64/OVMF_VARS.4m.fd vars.fd

swtpm socket --tpm2 --tpmstate dir=tpmstate --ctrl type=unixio,path=swtpm.sock \
  --flags not-need-init,startup-clear --daemon

qemu-system-x86_64 -machine q35 -m 512 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/x64/OVMF_CODE.4m.fd \
  -drive if=pflash,format=raw,file=vars.fd \
  -drive format=raw,file=fat:rw:esp \
  -chardev socket,id=chrtpm,path=swtpm.sock \
  -tpmdev emulator,id=tpm0,chardev=chrtpm \
  -device tpm-crb,tpmdev=tpm0 \
  -display none -serial stdio -net none
```

## Research & Educational Disclaimer

This repository is published strictly for academic security research, platform firmware auditing, and low-level virtualization engineering. The techniques and code presented here demonstrate systemic limitations within current firmware-level measured boot orchestration. All experimentation should be conducted solely on authorized hardware.
