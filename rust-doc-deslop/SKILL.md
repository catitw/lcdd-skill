---
name: rust-doc-deslop
description: "Use when remediating, cleaning up, or deslopping AI-generated comments and outdated docstrings in existing (brownfield) Rust codebases. Requires a mandatory module/directory/file path argument. Classifies comments into pure slop (delete), bad-code deodorant (refactor into named private helpers), unformalized safety (upgrade to // SAFETY:), and genuine invariants (preserve). Enforces regression-safe verification via cargo test."
---

# Rust Doc Deslop (LCDD Phase 3: Brownfield Remediation)

Clean up noisy AI comments, remove redundant code explanations, and upgrade loose invariant notes in existing Rust modules without changing runtime behavior.

---

## 0. Mandatory Scope Gate (No Blanket Sweeps)

**A target module, directory, or file path is STRICTLY REQUIRED.**

- **Allowed**:
  - Module path: `src/network/codec` or `crates/engine/src/state.rs`
  - Directory: `crates/storage/`
- **BANNED**:
  - Running without arguments against the whole repository (e.g. bare `/rust-doc-deslop`).

> **Rule**: If no path is provided, STOP immediately and ask the user which module or file to deslop. Do not guess or touch the entire workspace at once.

---

## 1. Comment Triage Taxonomy (The 5 Classes)

Scan every comment in the target scope and classify it before taking action:

| Category | Typical Pattern | Action |
| :--- | :--- | :--- |
| **A: Pure AI Slop** | • **The Narrator**: `// Loop through items`<br>• **The Translator**: `i += 1; // Increment i`<br>• **The Tutorial**: `// Step 1: parse`, `// Step 2: send`<br>• **Tautology**: `/// Gets user.` above `fn get_user` | **Physical Deletion**: Delete comment and any blank lines it leaves behind. |
| **B: Code Deodorant** | A comment explaining a convoluted 4-line conditional or complex iterator chain. | **Micro-Refactor**: Extract the block into a named private helper function or clear local boolean, then delete the comment. |
| **C: Informal Soundness** | Loose comments near `unsafe {}` or `unsafe fn` (e.g. `// bounds checked above`, `// safe pointer deref`). | **Formalize**: Upgrade to standard `// SAFETY:` invariant proof or `/// # Safety` section. |
| **D: Real Invariants & Workarounds** | Hardware quirks, OS-specific workarounds, math formulas, complex atomic memory-ordering rationale. | **Preserve & Tighten**: Keep the context intact. Ensure bug workarounds link to an external tracking URL/issue. |
| **E: Doc Drift (Rot)** | Doc comments referencing obsolete parameter names, dropped fields, or wrong return semantics. | **Realign**: Fix doc text to match current type signatures, anchoring types with intra-doc links (`[`TypeName`]`). |

---

## 2. Modes of Operation

### Mode 1: Audit-Only (`--review`)
When invoked with `--review`, do NOT edit files. Produce a structured triage summary:

```markdown
### Deslop Audit Report: <module_path>
- **Files Scanned**: 3
- **Pure Slop Removed (Cat A)**: 18 occurrences
- **Candidates for Extraction (Cat B)**: 2 (e.g. `src/foo.rs:42` complex timeout check)
- **Safety Proofs Formalized (Cat C)**: 1 (e.g. `src/bar.rs:88` unsafe slice conversion)
- **Invariants Preserved (Cat D)**: 2 (e.g. Quake inverse square root, Linux TCP bug workaround)
- **Doc Drift Corrected (Cat E)**: 1 (stale error variant reference)
```

### Mode 2: Apply (Default)
Perform surgical edits directly, following the safety workflow below.

---

## 3. Execution Workflow (Step-by-Step)

1. **Lock Baseline Behavior**:
   Run the module-targeted test suite before touching anything:
   ```bash
   cargo test --lib <module_path>
   ```
   If tests are failing before you start, STOP and report the baseline failure first.

2. **Pass 1: Delete Pure AI Slop (Category A)**:
   - Strip out all narrative, translation, and tutorial comments from function bodies.
   - Clean up orphan blank lines left behind by deleted comments.

3. **Pass 2: Extract Code Deodorant (Category B)**:
   - Identify comments explaining what complex code does.
   - Turn the logic into a self-documenting private helper:
     ```rust
     // BEFORE:
     // Check if transaction has expired and has not been acknowledged
     if tx.timestamp.elapsed() > TIMEOUT && !tx.is_ack() { ... }

     // AFTER:
     if is_stale_unacknowledged(&tx) { ... }

     #[inline]
     fn is_stale_unacknowledged(tx: &Transaction) -> bool {
         tx.timestamp.elapsed() > TIMEOUT && !tx.is_ack()
     }
     ```

4. **Pass 3: Upgrade Safety Proofs (Category C)**:
   - Find all `unsafe {}` blocks in the module.
   - Ensure each has a dedicated `// SAFETY:` comment directly above it, listing why preconditions are met.

5. **Pass 4: Realign Public Docs (Category E)**:
   - Audit public items: replace raw text mentions with `[`IntraDocLinks`]`.
   - Remove Javadoc `@param` or `@return` tags; replace with RFC 1574 `# Errors` / `# Panics` sections if missing.

---

## 4. Verification & Quality Gates

After edits, run the verification ladder:

```bash
# 1. Compilation & type check
cargo check

# 2. Module regression tests
cargo test --lib <module_path>

# 3. Soundness & clippy validation
cargo clippy -- -D clippy::undocumented_unsafe_blocks

# 4. Intra-doc links verification
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps
```

All gates must be 100% green. If any test or lint fails, fix the regression immediately or roll back that specific edit.
