# My Escrow Project — Learnings and Notes

This project is my implementation and learning exercise based on the Anchor Escrow example. Below I narrate what I learned, the problems I solved, and where to look in the codebase.

## Short summary (in my voice)
I built and iterated on a Solana Anchor escrow program that implements ERC‑style token swaps: a maker creates an offer by depositing token A into a vault (an ATA owned by a PDA), and a taker accepts by sending token B to the maker. Along the way I learned Anchor account constraints, PDAs, token program interactions, and how to test both in Rust (LiteSVM) and TypeScript (Solana Kite).

## Key learnings

- Anchor program structure and macros
  - How #[program] and Context<> connect instructions to handler functions — see [programs/escrow/src/lib.rs](programs/escrow/src/lib.rs).
  - Writing instruction handlers as pure functions and using typed Account structs (e.g., `MakeOffer`, `TakeOffer`) in [programs/escrow/src/handlers](programs/escrow/src/handlers).

- PDAs & seeds
  - Creating and using PDAs for offer accounts and their associated vault ATAs. Example: the `offer` PDA seeded with `["offer", id.to_le_bytes()]` in [programs/escrow/src/handlers/make_offer.rs](programs/escrow/src/handlers/make_offer.rs).
  - Utility helpers for PDA generation used in tests: [`generate_offer_id`](programs/escrow/src/escrow_test_helpers.rs) and test PDAs.

- Token operations and account ownership
  - Moving tokens between ATAs and a vault owned by the Offer PDA using Anchor + anchor_spl tokens. See shared transfer helper in [programs/escrow/src/handlers/shared.rs](programs/escrow/src/handlers/shared.rs) and the make/take handlers: [`make_offer`](programs/escrow/src/handlers/make_offer.rs) and [`take_offer`](programs/escrow/src/handlers/take_offer.rs).
  - Using associated token accounts (ATAs) with `associated_token::get_associated_token_address` patterns.

- Anchor account constraints and initialization
  - How to use `#[account(init, payer = ..., seeds = [...], bump)]` and `associated_token` constraints; pattern visible in [programs/escrow/src/handlers/make_offer.rs](programs/escrow/src/handlers/make_offer.rs).

- Error handling and custom errors
  - Defining Anchor error codes for clear testable failure modes in [programs/escrow/src/error.rs](programs/escrow/src/error.rs).

- Testing strategy
  - Fast Rust unit tests using LiteSVM for program logic and state transitions — setup in [programs/escrow/src/escrow_test_helpers.rs](programs/escrow/src/escrow_test_helpers.rs) and tests in [programs/escrow/src/tests.rs](programs/escrow/src/tests.rs).
  - TypeScript integration tests with Solana Kite and the generated client in [tests/escrow.test.ts](tests/escrow.test.ts) and helpers in [tests/escrow.test-helpers.ts](tests/escrow.test-helpers.ts).

- Tooling & workflow
  - Anchor build/test (`anchor build`, `anchor test`) and generating TypeScript clients (`npx create-codama-clients`) as choreographed in [package.json](package.json) and [Anchor.toml](Anchor.toml).
  - Managing warnings and toolchain quirks (e.g., SBF/Cargo/Anchor versions listed in [README.md](README.md) and the CHANGELOG).

## ⛓️ Cross-Program Invocation (CPI) & Vault Mastery

The core "magic" of this Escrow program is its ability to talk to other programs on-chain. Here is what I mastered:

### 1. The CpiContext Pattern
I learned that to move tokens, my program must "ask" the SPL Token Program to do it. I implemented `CpiContext` to bundle the necessary accounts and instructions:
* **Shared Helpers:** Created a `transfer_tokens` helper in `shared.rs` to keep the code DRY (Don't Repeat Yourself).
* **Instruction Passing:** Learned how to pass the `token_program` account and build `Transfer` structs for the CPI.

### 2. Signing with Seeds (PDA Power)
Since a PDA doesn't have a private key, it cannot sign a transaction normally.
* **Problem:** How does the "Vault" send tokens to the Taker?
* **Solution:** I used `CpiContext::new_with_signer`. By passing the seeds (e.g., `b"offer"`, `maker_key`, `id`) and the `bump`, the Solana runtime verifies the seeds and allows my program to sign on behalf of the Vault.

### 3. Anchor Account Constraints
I used Anchor’s powerful macros to automate security checks that would otherwise require dozens of lines of manual code:
- `has_one = maker`: Ensures only the original creator can cancel an offer.
- `constraint = ...`: Custom logic to verify that the taker is sending the correct amount and type of token.

## Problems I solved / gotchas

- Ensuring vault ATAs are owned by the PDA so the program can sign with PDA seeds for withdrawals.
- Carefully encoding instruction discriminators and instruction data for manual instruction building in tests (`get_make_offer_discriminator` etc. in [programs/escrow/src/escrow_test_helpers.rs](programs/escrow/src/escrow_test_helpers.rs)).
- Handling insufficient-funds flows in both make and take paths and asserting errors in tests (see tests for `InsufficientMakerBalance` / `InsufficientTakerBalance`).
- Managing anchor_spl and Solana library versions to avoid unexpected cfg warnings (see build artifacts and warnings in the project `target/` outputs).

## Files / places to inspect (quick links)
- Program entry & instruction list: [programs/escrow/src/lib.rs](programs/escrow/src/lib.rs)  
- MakeOffer handler: [programs/escrow/src/handlers/make_offer.rs](programs/escrow/src/handlers/make_offer.rs)  
- TakeOffer handler: [programs/escrow/src/handlers/take_offer.rs](programs/escrow/src/handlers/take_offer.rs)  
- Shared transfer helper: [programs/escrow/src/handlers/shared.rs](programs/escrow/src/handlers/shared.rs)  
- Errors: [programs/escrow/src/error.rs](programs/escrow/src/error.rs)  
- Test helpers (Rust): [programs/escrow/src/escrow_test_helpers.rs](programs/escrow/src/escrow_test_helpers.rs) — see [`generate_offer_id`](programs/escrow/src/escrow_test_helpers.rs)  
- Integration tests (TS): [tests/escrow.test.ts](tests/escrow.test.ts) and [tests/escrow.test-helpers.ts](tests/escrow.test-helpers.ts)  
- Cargo settings: [programs/escrow/Cargo.toml](programs/escrow/Cargo.toml)  
- Project README (original): [README.md](README.md)

## Tips for future work
- Add more granular tests around edge cases: reusing offer IDs, partial transfers, and race conditions.
- Consider adding event logs for easier off‑chain indexing.
- Add CI checks to ensure Anchor/solana versions remain pinned and tests run on the same toolchain.

