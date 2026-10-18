## Summary
Provide a concise overview of the technical modifications, crate additions, or firmware protocol updates introduced in this pull request.

## Target Subsystems
- [ ] `crates/ib-loader`: Core UEFI application entry point
- [ ] `crates/ib-tcg2`: Synthetic `EFI_TCG2_PROTOCOL` engine
- [ ] `crates/ib-tpm-crb`: Bare-metal CRB MMIO driver
- [ ] `crates/ib-tcglog`: TCG event log serialization
- [ ] `crates/ib-config`: Configuration structure definitions
- [ ] `host/ib-install`: Windows deployment utility
- [ ] `host/ib-tcg-dump`: Clean TCG baseline acquisition tool

## Architecture & Verification
1. Has this change been validated on physical hardware or under QEMU/OVMF with swtpm?
2. Does this modification maintain strict zero-trace NVRAM hygiene?
3. Have all memory bounds and unsafe blocks been documented with safety rationales?

## Related Issues
Closes #[issue-number]
