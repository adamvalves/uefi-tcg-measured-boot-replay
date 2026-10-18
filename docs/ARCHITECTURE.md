# Architectural Specification: UEFI Measured Boot Replay & TCG2 Protocol Emulation

## 1. System Foundations & Trust Boundary Analysis

Measured Boot on modern x86_64 platforms relies on a chain of cryptographic trust anchored in hardware:
- **Intel Boot Guard (ACM) / AMD Platform Secure Boot (PSB)**: Authenticates the Initial Boot Block (IBB) in the SPI flash memory prior to reset vector release.
- **Pre-EFI Initialization (PEI) & Driver Execution Environment (DXE)**: Measures subsequent firmware volumes, option ROMs, boot loaders, and configuration profiles into the Trusted Platform Module (TPM).
- **TPM 2.0 PCR Banks**: Cryptographic accumulators updated exclusively via the commutative extension formula:
  $$\text{PCR}_{\text{new}} = \mathcal{H}(\text{PCR}_{\text{old}} \mathbin{\Vert} \text{Measurement})$$
- **Operating System Loader (winload.efi) & Kernel (ntoskrnl.exe)**: Measures kernel components, ELAM drivers, BitLocker state, and code integrity policies.

The fundamental weakness exploited by this architecture resides in the decoupling between hardware-level measurement enforcement and firmware setup policy:
The hardware roots of trust (ACM/PSB) verify only that the firmware image is signed by the OEM. They do not enforce whether the firmware enables or suppresses the TPM 2.0 measurement engine during execution.

```
+-----------------------------------------------------------------------------------+
| Intel Boot Guard (ACM) / AMD Platform Secure Boot (PSB)                           |
| Hardware Root of Trust: Verifies OEM Key Digest -> Executes Signed SEC Core       |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| UEFI Firmware PEI / DXE Dispatcher                                                |
| Evaluates Setup Variable: TpmSpecVersion Offset (0x1B on AMI Aptio)               |
|                                                                                   |
| If Setup[0x1B] == 0x01 (TCG 2.0): Firmware initializes TPM2 and extends PCRs.   |
| If Setup[0x1B] == 0x00 (TCG 1.2): Firmware SUPPRESSES TPM2 PEI/DXE extends!      |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| TPM 2.0 Device Physical State (via CRB MMIO @ 0xFED40000)                         |
| Device is online, fully functional, and ready for commands.                       |
| SHA-256 PCR Banks 0 through 7 remain at initial reset value: 0x0000...0000        |
+-----------------------------------------------------------------------------------+
```

---

## 2. Command Response Buffer (CRB) MMIO Interface Driver

The TPM 2.0 specification defines two primary physical transport layers:
1. **FIFO (TIS)**: Traditional byte-oriented interface via hardware localities.
2. **Command Response Buffer (CRB)**: High-speed physical memory-mapped buffer interface utilized by modern client platforms (Intel PTT and AMD fTPM).

The driver implemented in `crates/ib-tpm-crb` communicates directly with the hardware through memory-mapped I/O registers located at base address `0xFED40000`.

### Locality Request & Control State Transitions

To transmit commands, the driver must assert and verify locality 0:

```
        +--------------+
        |  Idle State  |
        +--------------+
               |
               | Write CRB_LOC_CTRL.requestAccess = 1
               v
        +------------------+
        |  Request Locality |
        +------------------+
               |
               | Poll CRB_LOC_STS until locAssigned == 1
               v
        +------------------+
        | Locality Active  |
        +------------------+
               |
               | Write CRB_CTRL_REQ.cmdReady = 1
               v
        +------------------+
        | Ready for Buffer |
        +------------------+
```

### Command Dispatch & Execution Engine

```rust
pub unsafe fn submit_command(&mut self, cmd: &[u8], rsp: &mut [u8]) -> Result<usize, CrbError> {
    // 1. Ensure TPM is in Ready state
    if !self.is_ready() {
        self.set_ready()?;
    }

    // 2. Write raw frame into physical command buffer
    let cmd_buf_addr = self.get_cmd_buffer_address();
    core::ptr::copy_nonoverlapping(cmd.as_ptr(), cmd_buf_addr, cmd.len());

    // 3. Trigger hardware start
    self.write_reg(CRB_CTRL_START, 1);

    // 4. Poll completion status with timeout
    while self.read_reg(CRB_CTRL_START) & 1 != 0 {
        core::hint::spin_loop();
    }

    // 5. Read back physical response buffer
    let rsp_buf_addr = self.get_rsp_buffer_address();
    let rsp_len = self.read_response_size(rsp_buf_addr);
    core::ptr::copy_nonoverlapping(rsp_buf_addr, rsp.as_mut_ptr(), rsp_len);

    Ok(rsp_len)
}
```

---

## 3. TCG Event Log Reconstruction & PCR Extension Engine

A genuine measured boot is characterized by the sequential hashing and accumulation of events recorded within the EFI TCG2 Event Log.

### Event Format Serialization (`TCG_PCR_EVENT2`)

Each entry in a crypto-agile TCG 2.0 event log is defined as:
```c
typedef struct {
    UINT32            PCRIndex;
    UINT32            EventType;
    TPML_DIGEST_VALUES Digests;
    UINT32            EventSize;
    UINT8             Event[1];
} TCG_PCR_EVENT2;
```

### PCR Sequential Replay Loop

When `ib-loader.efi` executes during Stage 2, it deserializes the captured `tcglog.ib` baseline snapshot and drives the hardware TPM:

```
+-------------------------------------------------------------------+
| Baseline Event Log Snapshot (tcglog.ib)                          |
+-------------------------------------------------------------------+
           |
           | Iterate Record 0..N
           v
+-------------------------------------------------------------------+
| Filter: PCR Index within Target Scope (PCR 0 through PCR 7)      |
+-------------------------------------------------------------------+
           |
           v
+-------------------------------------------------------------------+
| Extract SHA-256 Digest (Algorithm ID: 0x000B)                    |
+-------------------------------------------------------------------+
           |
           v
+-------------------------------------------------------------------+
| Assemble TPM2_CC_PCR_Extend Command Packet                       |
|   - Handle: TPMI_DH_PCR (PCR Index)                              |
|   - Auth: TPM_RS_PW Password Session (Empty)                     |
|   - Value: 32-byte Event Digest                                  |
+-------------------------------------------------------------------+
           |
           v
+-------------------------------------------------------------------+
| Transmit over CRB MMIO -> Hardware Extends Internal Register     |
+-------------------------------------------------------------------+
```

Because PCR extension is strictly deterministic and order-dependent, replaying the genuine event log records in chronological order generates the exact 256-bit hash values produced by the platform firmware during an unmodified boot.

---

## 4. Synthetic `EFI_TCG2_PROTOCOL` Displacement

To prevent subsequent components (such as `bootmgr.efi`, `winload.efi`, and anti-cheat kernel drivers) from detecting discrepancies or discovering an uninitialized log, `ib-loader` implements and registers a displacement `EFI_TCG2_PROTOCOL`.

### Protocol VTable Structure

```c
typedef struct _EFI_TCG2_PROTOCOL {
    EFI_TCG2_GET_CAPABILITY               GetCapability;
    EFI_TCG2_GET_EVENT_LOG                GetEventLog;
    EFI_TCG2_HASH_LOG_EXTEND_EVENT        HashLogExtendEvent;
    EFI_TCG2_SUBMIT_COMMAND               SubmitCommand;
    EFI_TCG2_GET_ACTIVE_PCR_BANKS         GetActivePcrBanks;
    EFI_TCG2_SET_ACTIVE_PCR_BANKS         SetActivePcrBanks;
    EFI_TCG2_GET_RESULT_OF_SET_ACTIVE_PCR_BANKS GetResultOfSetActivePcrBanks;
} EFI_TCG2_PROTOCOL;
```

### Handler Interception Policies

1. **`GetEventLog`**:
   - Returns a pointer to the replayed clean event log buffer previously loaded from `tcglog.ib`.
   - Populates `EventLogLocation`, `EventLogLastEntry`, and signals `EventLogTruncated = FALSE`.
2. **`HashLogExtendEvent`**:
   - For calls targeting PCR 0 through 7: The synthetic protocol safely absorbs the event, appends it to the final events table if necessary, but drops hardware extension requests to avoid mutating the pristine replayed PCR state.
   - For OS-level dynamic measurements (e.g. BitLocker volume unlock events, PCR 11/12): Passes through directly to the underlying hardware.
3. **`SubmitCommand`**:
   - Forwards general TPM 2.0 commands directly to the CRB interface driver.

---

## 5. Non-Volatile Memory (NVRAM) Artifact Sanitization

The use of the Red Hat `shim` bootloader leaves characteristic audit artifacts in UEFI non-volatile memory variables. Before transferring execution to the Windows Boot Manager, `ib-loader` programmatically purges these variables using `SetVariable()` with a size of zero:

| Variable Name | Vendor GUID | Purpose | Action Taken |
| :--- | :--- | :--- | :--- |
| `L"MokList"` | `605DAB50-E046-4300-ABB6-3DD810DD8B23` | Enrolled MOK public certificates | Purged (`DataSize = 0`) |
| `L"MokListRT"` | `605DAB50-E046-4300-ABB6-3DD810DD8B23` | Runtime copy of MOK list | Purged (`DataSize = 0`) |
| `L"SbatLevel"` | `605DAB50-E046-4300-ABB6-3DD810DD8B23` | Secure Boot Advanced Targeting policy | Purged (`DataSize = 0`) |
| `L"MokOwner"` | `605DAB50-E046-4300-ABB6-3DD810DD8B23` | Machine Owner UUID | Purged (`DataSize = 0`) |
| `L"Setup"` | `EC87D643-EBA4-4BB5-A1E5-3F3E36B20DA9` | Platform Setup NVRAM (Offset 0x1B) | Restored to `0x01` (TCG 2.0) |

Following this purge, no NVRAM telemetry indicates that a custom shim or MOK was ever loaded on the system.
