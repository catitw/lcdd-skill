---
name: rust-silent-coding
description: "Use when writing, implementing, refactoring, or fixing Rust code, functions, or modules. Enforces clean, self-documenting implementation with ZERO inline narrative comments, eliminates AI code-restatement slop, and restricts inline comments to a strict soundness/invariants whitelist."
---

# Rust Silent Coding (LCDD Phase 1)

Enforce clean, silent implementation when authoring or modifying Rust code. The code must explain itself through expressive types, domain-accurate identifiers, and small, pure helper functions.

## Core Rule: Zero Narrative Inline Comments

Never narrate, translate, or explain ordinary control flow inside function bodies.

### Banned Patterns (Immediate Rejection)

1. **The Narrator** — Announcing standard language constructs:
   ```rust
   // BAD:
   // Loop through the items
   for item in items { ... }
   // Check if user is authenticated
   if user.is_authenticated() { ... }
   // Return early if error
   if err { return Err(...); }
   ```
2. **The Translator** — Restating code statements in natural language:
   ```rust
   // BAD:
   i += 1; // Increment counter
   buffer.clear(); // Clear the buffer
   return Ok(response); // Return successful response
   ```
3. **The Step-by-Step Tutorial** — Writing numbered progress markers for standard logic:
   ```rust
   // BAD:
   // Step 1: Parse incoming payload
   let payload = parse(raw)?;
   // Step 2: Validate signatures
   validate(&payload)?;
   // Step 3: Dispatch message
   dispatch(payload).await?;
   ```
4. **The Obvious Identifier Restater**:
   ```rust
   // BAD:
   let user_id = get_user_id(); // Get user id
   let is_expired = token.is_expired(); // Check if token expired
   ```

---

## Replacement Disciplines (Make Code Self-Documenting)

When tempted to write an inline comment to explain complex logic:

1. **Extract a Named Private Helper**:
   ```rust
   // BAD:
   // Check if the packet timestamp is within the allowable 500ms jitter window
   if current_ts >= packet_ts && (current_ts - packet_ts) <= Duration::from_millis(500) { ... }

   // GOOD:
   if is_within_jitter_window(current_ts, packet_ts) { ... }

   fn is_within_jitter_window(current: Instant, packet: Instant) -> bool {
       current >= packet && (current - packet) <= Duration::from_millis(500)
   }
   ```

2. **Introduce Expressive Local Bindings**:
   ```rust
   // BAD:
   // Ensure channel is open and client has write quota remaining
   if state == State::Connected && remaining_quota > 0 { ... }

   // GOOD:
   let can_send = state == State::Connected && remaining_quota > 0;
   if can_send { ... }
   ```

3. **Use Typestates / Enums Over Flag Variables**:
   Encode state and invariants into Rust's type system rather than commenting on ambiguous booleans or integers.

---

## Strict Inline Comment Whitelist

Inline comments (`//`) inside function bodies are **STRICTLY FORBIDDEN** unless they match one of the following 4 categories:

### 1. `// SAFETY:` Invariant Proofs (Mandatory for Unsafe)
Every `unsafe {}` block MUST have a `// SAFETY:` comment directly preceding it, proving why undefined behavior is impossible at this exact call site:
```rust
// SAFETY: `index` was verified to be less than `self.len` by the guard above,
// and `self.data` is guaranteed properly aligned and non-null by type invariants.
unsafe { *self.data.as_ptr().add(index) }
```

### 2. External Bug / Hardware / Quirks Workarounds (Mandatory URL)
Workarounds for upstream compiler bugs, external protocol errata, or OS-specific quirks MUST reference an external tracking URL:
```rust
// WORKAROUND: Linux kernel TCP zero-window probe bug on older kernels.
// See: https://github.com/rust-lang/rust/issues/12345
let _ = stream.set_nodelay(true);
```

### 3. Non-Obvious Mathematical or Algorithmic Invariants
Citing specific formulas or subtle numerical stability reasons:
```rust
// Fast inverse square root constant approximation (Quake III algorithm).
// Relative error bounded within 0.175%.
i = 0x5f3759df - (i >> 1);
```

### 4. Concurrency & Memory-Ordering Rationale
Explaining why a specific `Ordering` (e.g. `Acquire`/`Release`) guarantees memory synchronization across threads:
```rust
// Synchronization: Release pairs with Acquire in `consumer_thread` to ensure
// all buffer writes prior to this store are visible before state becomes Active.
self.state.store(ACTIVE, Ordering::Release);
```

---

## Verification

Before finishing code edits in this phase, verify:
1. `cargo check` passes with zero warnings.
2. `cargo test` passes.
3. Review git diff: Ensure NO narrative `//` comments exist inside any touched function body.
