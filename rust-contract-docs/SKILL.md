---
name: rust-contract-docs
description: "Use when adding or reviewing documentation, writing doc comments (/// or //!), or preparing public Rust APIs for PR/release. Supports `--diff` (or `--diff <base>`) to strictly scope documentation only to touched/modified public items. Enforces RFC 1574 contract sections (# Errors, # Panics, # Safety, runnable # Examples), mandates intra-doc links, enforces module-level //! architecture scope, and verifies via doctests and clippy."
---

# Rust Contract Documentation (LCDD Phase 2)

Inject crisp, black-box contractual documentation into Rust modules and public APIs. Document for the external consumer and the future auditor, not the line-by-line implementer.

---

## 0. Scope Modes: `--diff` vs Explicit Path

To prevent scope creep and bloated diffs, use targeted scoping:

### Option 1: `--diff [base]` (Strictly Scoped to Recent Changes)
When invoked with `--diff` (e.g. `/skill:rust-contract-docs --diff` or `--diff main`):

1. **Detect Touched Files**:
   ```bash
   git diff --name-only ${base:-HEAD} | grep '\.rs$'
   ```
   Combine with untracked files from `git status --porcelain`.
2. **Lock Scoped Items**:
   - **ONLY** document `pub` items (functions, structs, enums, traits, methods) that were **added or whose signatures/implementations were modified** in that diff.
   - **DO NOT TOUCH UNMODIFIED ITEMS**: Even if an adjacent legacy function in the same file lacks documentation, **LEAVE IT ALONE**. Do not expand diffs without explicit permission.
3. **Targeted Doctests**:
   Run doctests only for the touched modules/crates to keep feedback cycles instant.

### Option 2: Explicit Module / Path Scope
If a path is explicitly specified (e.g. `/skill:rust-contract-docs src/network/codec.rs`), audit and document all public items within that specific file or module.

---

## 1. Anti-Hallucination: Ground Terminology via Intra-Doc Links

Agents frequently invent conceptual role names (e.g. `PipelineWorkerCoordinator`) not present in the code.

**The Rule**:
Whenever mentioning a domain entity, function, or concept in documentation, **MUST anchor it to an actual code symbol using Rustdoc intra-doc links**:
- Use `[`MyStruct`]` or `[`crate::module::MyType`]`
- Use `[`MyTrait::method`]` or `[`Option::map`]`
- **BANNED**: Inventing generic, unanchored capital-letter role descriptions that do not exist as concrete types or traits in the workspace.

If an intra-doc link fails to resolve, `cargo doc` will immediately reject it as an unresolved link error.

---

## 2. The 4-Tier Comment Scope Hierarchy

Never blur documentation scopes:

| Scope | Token | Audience & Purpose | Forbidden Content |
| :--- | :--- | :--- | :--- |
| **Tier 1: Module / Crate** | `//!` | **System Architecture & Mental Model**: Module responsibility, lifecycle, state machine transitions, cross-type interactions. | **NO** trivial listing of exported structs/functions (rustdoc generates this automatically). |
| **Tier 2: Public Item Contract** | `///` | **Black-Box Caller Contract**: Preconditions, postconditions, error modes, panic conditions, safety requirements. | **NO** internal implementation details (e.g., "uses a binary heap internally"). **NO** `@param`/`@return` tags. |
| **Tier 3: Soundness Proof** | `// SAFETY:` | **Auditor Proof**: Exact justification why an `unsafe {}` block will not cause undefined behavior at this call site. | **NO** vague assertions (e.g. `// SAFETY: safe to call`). Must list verified preconditions. |
| **Tier 4: Subtle Invariant** | `//` | **Maintainer Context**: External bug workarounds with URLs, mathematical proofs, lock-ordering invariants. | **NO** narrative statements describing standard logic flows. |

---

## 3. RFC 1574 Public Item Documentation Standard

Every public function, method, and trait item MUST follow this section structure:

```rust
/// One-sentence summary in the active voice describing *what* the item accomplishes.
///
/// Optional detailed explanation expanding on caller expectations, domain context,
/// and ownership/borrowing semantics. Reference types via [`ItemName`].
///
/// # Errors
///
/// Mandatory if the function returns [`Result`]. Enumerate the exact domain conditions
/// under which `Err` variants are returned:
/// - Returns [`Error::NotFound`] if the target key is absent from the index.
/// - Returns [`Error::Io`] if flushing the backing storage fails.
///
/// # Panics
///
/// Mandatory if the function can panic, call `.expect()`, or perform out-of-bounds indexing:
/// - Panics if `capacity` is 0.
/// - Panics if called outside an active Tokio runtime context.
///
/// # Safety
///
/// Mandatory for all `unsafe fn`. Document every invariant and precondition the caller MUST uphold:
/// - `ptr` must be non-null and aligned to `align_of::<T>()`.
/// - The memory referenced by `ptr` must remain valid and un-aliased for the lifetime `'a`.
/// - Violating these guarantees causes undefined behavior.
///
/// # Examples
///
/// Mandatory for all primary crate exports. Must be a fully runnable doctest:
///
/// ```rust
/// use my_crate::TokenBucket;
///
/// # fn run() -> Result<(), Box<dyn std::error::Error>> {
/// let mut bucket = TokenBucket::new(10)?;
/// assert!(bucket.try_acquire()?);
/// # Ok(())
/// # }
/// # run().unwrap();
/// ```
```

### Doctest Rules:
1. **Always use `?`** for error handling in examples. Avoid `.unwrap()` unless specifically demonstrating a panic scenario.
2. **Hide boilerplate setup**: Use `# ` at the beginning of setup lines so the reader only sees relevant code, but `cargo test --doc` executes the full fixture.
3. **No pseudocode**: Do NOT use `...` or unresolvable placeholder crates. Every example must compile and pass.

---

## 4. Forbidden Doc Patterns

- **NO JavaDoc / TSDoc tags**: Strictly ban `@param`, `@return`, `@throws`, `@type`. Rust is strongly typed; the signature documents types.
- **NO tautological summaries**:
  ```rust
  // BAD:
  /// Creates a new user.
  pub fn new_user(...) -> User;

  // GOOD:
  /// Allocates an active user profile with initialized quotas and a default audit session.
  pub fn new_user(...) -> User;
  ```

---

## 5. Automated Verification Gates

Run all three gates before marking documentation complete:

```bash
# 1. Reject broken intra-doc links & missing doc warnings
RUSTDOCFLAGS="-D warnings" cargo doc --no-deps

# 2. Execute doctest examples (scoped or crate-wide)
cargo test --doc

# 3. Enforce contract sections & soundness comments
cargo clippy -- -D clippy::undocumented_unsafe_blocks -D clippy::missing_errors_doc -D clippy::missing_safety_doc -W clippy::missing_panics_doc -W clippy::doc_markdown
```

If any command fails, fix the code/docs immediately.
