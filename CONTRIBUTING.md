# Contributing Guidelines

Thank you for your interest in contributing to this low-level systems and firmware research repository.

## Contribution Standards

### 1. Code Quality & Formatting
- All Rust code must adhere strictly to `rustfmt` formatting standards (`cargo fmt --check`).
- Clippy warnings are treated as hard compilation errors (`deny`).
- Every `unsafe` block must be accompanied by an explicit `// SAFETY:` rationale explaining pointer provenance, alignment, and invariant preservation.

### 2. Documentation & Tone
- All commits, pull requests, and documentation must be written in formal, authoritative English.
- Use accurate systems engineering terminology (e.g., "second-level address translation", "CRB physical MMIO registers", "SHA-256 PCR banks").
- **Zero Emojis**: Do not use emojis in commit messages, pull requests, issue templates, or source code comments.

### 3. Git Commit Conventions
Follow Conventional Commits:
- `feat(crb): implement tpm locality status polling`
- `fix(tcg2): correct event log last entry pointer calculation`
- `docs(arch): add crb register layout specifications`
- `refactor(loader): streamline nvram cleanup routines`

### 4. Author Identity
All contributions must be signed off in accordance with the Developer Certificate of Origin (DCO).
