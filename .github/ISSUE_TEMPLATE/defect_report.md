---
name: Defect Report
about: Create a structured report to reproduce and fix a software defect
title: "[BUG] "
labels: ["defect", "triage"]
assignees: ["adamvalves"]
---

## Description
A technical summary of the observed defect.

## Target Platform & Hardware Environment
- Processor: [e.g. AMD Ryzen 9 7950X / Intel Core i9-13900K]
- Motherboard: [e.g. ASUS ROG STRIX X670E-E]
- BIOS / Firmware Revision: [e.g. Version 1807, AMI Aptio V]
- TPM Specification: [Discrete TPM 2.0 / Intel PTT / AMD fTPM]
- TPM Interface Type: [Command Response Buffer (CRB) / FIFO (TIS)]
- Setup Offset: [e.g. 0x1B]

## Steps to Reproduce
1. Execute `ib-tcg-dump.exe` under clean boot.
2. Stage boot files via `ib-install.exe`.
3. Reboot and enroll MOK key.
4. Observe boot failure or attestation divergence.

## Observed Behavior
Provide exact error code, UEFI console output, or NTSTATUS failure.

## Expected Behavior
Precise description of expected execution flow and attestation state.

## Log Output
```
Insert relevant serial console, TBS, or debug logs here
```
