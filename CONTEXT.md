# CONTEXT — Security & Audit Policies

> **Audience:** This file is written to be consumed by an automated **AI audit
> agent** as well as human reviewers. It enumerates the known failure classes in
> this codebase (`/contracts` and `/crates`) and the mandatory token-handling
> policies any current or future contract in this repository **must** satisfy.
> Each policy is phrased as a checkable rule with rationale, detection guidance,
> and the expected mitigation so it can be mechanically verified.
>
> **Scope note:** The `tornado_classic` mixer in this repo today is
> commitment-only (it inserts Merkle leaves and verifies withdrawal proofs but
> does **not** yet move any SEP-41/`token` balances). The token policies below
> are **forward-looking, binding requirements**: the moment a contract here
> begins transferring fungible tokens (e.g. a denominated mixer, vault, or
> shares contract), these rules apply and must be audited.

## A. General code failure points (verifier & contracts)

These are recurring failure classes the audit agent should flag wherever they
appear. They are written as *general ideas* rather than fixed line numbers,
because the code will evolve.

1. **Field inversion without a zero check.**
   Any call to `.inverse()` / modular inverse (e.g. in `verifier.rs`,
   `sumcheck.rs`, `shplemini.rs`) must be preceded by an explicit
   `is_zero()` guard or be performed through a checked/batched inverse that
   returns an error. Attacker-influenced public inputs or challenges can drive a
   denominator to zero. **Rule:** no unguarded inverse on any value derived from
   untrusted proof data, public inputs, or transcript challenges.

2. **Slice / array indexing without bounds reasoning.**
   Every slice range (`a..b`), `clone_from_slice`, `copy_into_slice`, and
   direct `[idx]` access must be provably in-bounds for **all** valid and
   adversarial inputs. Pay special attention to ranges parameterised by
   `log_n` / `log_circuit_size` versus the fixed `CONST_PROOF_SIZE_LOG_N`, and
   to source-vs-destination length mismatches in `clone_from_slice`. **Rule:**
   only the populated prefix `0..log_n-1` of variable-length arrays may be read;
   never read the uninitialised tail.

3. **Unvalidated sizes and parameters parsed from untrusted bytes.**
   Values like `log_circuit_size`, `circuit_size`, `public_inputs_size`, and any
   length prefix read in `utils.rs` deserialisers must be range-checked
   immediately after parsing (e.g. `0 < log_circuit_size <= CONST_PROOF_SIZE_LOG_N`).
   **Rule:** reject out-of-range structural parameters with a typed error before
   they are used as loop bounds or indices.

4. **Integer overflow / underflow.**
   Loop counters and offsets (e.g. `idx += 32` over `public_inputs`) must not be
   able to overflow their type. Prefer `u64` for byte offsets, use
   `checked_add` / `saturating_add` for accumulators, and validate that input
   lengths are exact multiples of the element stride before iterating.

5. **`unwrap()` / `expect()` / `panic!` in production paths.**
   Panics in on-chain code abort the transaction and can become a
   denial-of-service vector or mask logic errors. **Rule:** non-test code must
   return typed errors (`Result<_, MixerError>` / `Error`) instead of
   `unwrap`/`expect`/indexing panics. `unwrap` is acceptable only in `#[cfg(test)]`.

6. **Silently collapsed / generic errors.**
   Mapping every failure to a single generic error (e.g. all storage failures to
   `VkNotSet`, or all batch-inverse failures to a generic string) hides root
   causes. **Rule:** distinguish "absent" from "corrupt/invalid", and give
   proof-integrity failures (e.g. challenge collides with an interpolation point)
   a distinct error from infrastructure failures.

7. **Fiat–Shamir transcript completeness.**
   The transcript (`transcript.rs`) must absorb **every** proof element and
   public input that the soundness of the scheme depends on, in the canonical
   order, before deriving each challenge. **Rule:** any proof field that is used
   later but never hashed into the transcript is a soundness bug (verification
   bypass). The audit agent should cross-check the set of absorbed elements
   against the set of consumed elements.

8. **Off-by-one and boundary logic.**
   Merkle insertion loops, frontier indexing, and tree-depth bounds
   (`mixer.rs`, `TREE_DEPTH`, `MAX_LEAVES`) must be checked at the first and last
   iteration. Confirm `zeroes` has length `TREE_DEPTH + 1` and that `next_index`
   is checked against `MAX_LEAVES` *before* insertion.

9. **Constant-time / side-channel awareness.**
   Operations on secret-dependent data (where applicable) should avoid
   input-dependent branching/timing. `pow(exp)` and similar are low-risk only
   while exponents are public protocol constants — flag any change that feeds
   untrusted data into them.

10. **Replay & state-machine integrity (mixer).**
    Nullifier spent-checks must happen before any state mutation, the proof must
    bind to the *current* stored root, and the nullifier must be marked spent
    atomically with the effect. **Rule:** no withdrawal effect may occur before
    `verify_proof` succeeds and the nullifier is recorded.

## B. Token-handling policies (mandatory for any token-moving contract)

These policies exist because privacy/mixer and vault contracts are prime targets
for token-behaviour exploits. They apply to every SEP-41 / Stellar Asset
Contract (`token::Client`) the contract interacts with.

### B.1 Anti fee-on-transfer (deflationary / taxed) token policy

**Policy:** The contract MUST NOT assume that the amount it requested to
transfer equals the amount actually received. Fee-on-transfer (a.k.a.
deflationary or taxed) tokens deduct a fee on `transfer`, so a `transfer(amount)`
credits the recipient *less than* `amount`.

- **Why it matters here:** A denominated mixer relies on every deposit being
  *exactly* the fixed denomination so that all notes are fungible and withdrawals
  can pay out the full denomination. If a fee-on-transfer token under-credits the
  pool, the contract becomes under-collateralised and later withdrawals drain
  honest depositors' funds.
- **Detection (for the audit agent):**
  - Flag any code path that records a credited/deposited amount equal to the
    *input* argument rather than the *measured* balance delta.
  - Flag accounting that does `balance += amount` instead of
    `balance += (balance_after - balance_before)`.
- **Required mitigation (pick one and enforce):**
  1. **Balance-delta accounting:** read `token.balance(&this)` before and after
     the `transfer_from`/`transfer`, and use the measured delta as the credited
     amount; reject if the delta `!=` the expected denomination.
  2. **Allowlist of known well-behaved tokens** configured at deploy time, OR
  3. **Explicit rejection:** probe the token and refuse to operate if a transfer
     of `N` does not increase the contract balance by exactly `N`.
- **Invariant:** `received == requested` must be asserted (not assumed) for any
  fixed-denomination flow.

### B.2 Anti ERC-777 / reentrancy-callback token policy

**Policy:** The contract MUST NOT integrate tokens that perform **callbacks into
the caller/recipient during a transfer** (the ERC-777 `tokensReceived` /
`tokensToSend` hook pattern, or any token that re-enters the calling contract on
transfer).

- **Why it matters here:** Transfer-time callbacks enable **reentrancy**: a
  malicious token can re-enter `deposit`/`withdraw` mid-transfer, before
  nullifiers are marked spent or balances are settled, allowing double-spends or
  inconsistent state. While Soroban's execution model differs from the EVM,
  cross-contract calls can still re-enter, so the same threat class applies to
  any callback-on-transfer token.
- **Detection (for the audit agent):**
  - Flag integration with any token whose transfer triggers a call back into
    this contract or an arbitrary user-controlled address.
  - Flag any external call (including `token.transfer`) that occurs **before**
    all critical state is finalised.
- **Required mitigation:**
  1. **Checks-Effects-Interactions:** finalise all state (mark nullifier spent,
     update balances/root) **before** making the external token call.
  2. **Reentrancy guard:** maintain a transient/instance "locked" flag around any
     function that performs external token transfers.
  3. **Allowlist:** only integrate plain SEP-41 tokens with no transfer-time
     callback semantics; explicitly reject callback-capable token contracts.

### B.3 Token decimals bound policy (reject `< 6` and `> 18` decimals)

**Policy:** The contract MUST reject any token whose `decimals()` is **less than
6** or **greater than 18**. Only tokens with `6 <= decimals <= 18` are supported.

- **Why it matters here:**
  - **Too few decimals (`< 6`):** coarse smallest-unit granularity makes fixed
    denominations and fee math lossy, and amplifies rounding/precision attacks.
  - **Too many decimals (`> 18`):** large scaling factors (`10^decimals`) risk
    multiplication overflow in amount math and in any
    shares↔assets conversion, and exceed the de-facto interoperable range.
- **Detection (for the audit agent):**
  - Flag any contract that accepts an arbitrary token without reading
    `token.decimals()` and bounds-checking it at registration/deploy time.
- **Required mitigation:**
  - At the point a token is configured, call `decimals()` and
    `require(6 <= decimals && decimals <= 18)`, returning a typed error
    otherwise. Persist the validated `decimals` and never re-derive it from
    untrusted input later.

### B.4 Inflation-attack (first-depositor / share-price) awareness policy

**Policy:** Any contract that mints "shares" or otherwise tracks proportional
ownership against a pooled token balance MUST be hardened against the classic
**ERC-4626-style inflation / donation attack**, where an attacker who is the
first depositor inflates the share price via a direct token donation and then
steals a disproportionate share of subsequent deposits (victims' deposits round
down to zero shares).

- **Why it matters here:** Although the current mixer uses fixed denominations
  (one note = one fixed amount, no proportional shares), any future
  variable-amount pool, yield vault, or LP-style accounting in this repo is
  directly exposed. The audit agent should treat *any* `shares = amount * total_shares / total_assets`
  style formula as in-scope.
- **Detection (for the audit agent):**
  - Flag share-minting math where `total_assets` can be increased by a **direct
    token transfer** (donation) that bypasses the deposit accounting.
  - Flag the **first deposit** path where `total_supply == 0` and shares are set
    equal to `amount` with no floor / dead-shares protection.
  - Flag conversions that **round in the user's favour** or that can round a
    victim's minted shares down to zero.
- **Required mitigations (apply as appropriate):**
  1. **Internal balance tracking:** account assets from explicit deposit
     bookkeeping, not from `token.balance(self)`, so donations cannot move the
     share price. (This also composes with the B.1 balance-delta rule by tracking
     the *measured* deposit delta internally.)
  2. **Dead shares / initial mint to a burn address:** mint a small fixed amount
     of shares to a non-redeemable address (or the contract itself) on the first
     deposit so the price ratio cannot be cheaply manipulated.
  3. **Virtual shares/assets offset (ERC-4626 mitigation):** add a constant
     virtual offset to `total_shares` and `total_assets` in the conversion so the
     first depositor cannot set an extreme ratio.
  4. **Minimum deposit / minimum-shares-minted check:** reject deposits that
     would mint zero shares.
  - **Invariant:** no externally-triggerable, accounting-bypassing change to
    `total_assets` may alter the share price; the first depositor must not be
    able to make later depositors receive zero shares.

## C. How the audit agent should use this file

For each policy A.1–A.10 and B.1–B.4, the agent should: (1) locate the relevant
code paths in `/contracts` and `/crates`, (2) determine whether the required
invariant/mitigation is present, (3) report PASS / FAIL / NOT-APPLICABLE with the
file and line, and (4) for any FAIL, propose the concrete mitigation named above.
Token policies B.1–B.4 are **NOT-APPLICABLE only while no token transfers exist**;
the moment a `token::Client` transfer is introduced, they become FAIL-by-default
until the listed mitigation is implemented and verified.
