
# Compliance-Gated Stablecoin on Miden (real, private, enforceable)

## TL;DR

Enforce AML/KYC **inside the proved transaction** by anchoring compliance state in a **public registry account** and verifying **light proofs** (Merkle membership / absence, signatures or origin) in MASM. Pull the registry’s roots via **Foreign Procedure Invocation (FPI)**—a read-only cross-contract call—and gate transfers accordingly. This fits Miden’s **local execution & proving**, **public/private accounts & notes**, and **standard note scripts** model. ([0xmiden.github.io][1])

---

## What Miden gives you (and we’ll use)

* **Local transaction execution & proving**; operator verifies and updates state DBs. Great for private credentials with public enforcement. ([0xmiden.github.io][1])
* **Public & private accounts/notes** so you can keep user data private but publish compliance anchors (roots/epochs). ([0xmiden.github.io][1])
* **Standard/custom note scripts** (P2ID, P2IDR, SWAP) for spend-time conditions and reclaim powers. ([0xmiden.github.io][1])
* **Protocol Library procedures** with explicit **contexts** (Any / Account / Native / Auth / Faucet) so we know exactly where checks can run and where state can be mutated. ([0xmiden.github.io][2])
* **FPI**: `tx::execute_foreign_procedure` to read **foreign account** state inside your program’s execution trace. (Read-only—perfect for fetching allow/revoke roots & epochs). ([0xmiden.github.io][3])
* **Maps & storage slots** to hold roots/keys and rotate them (via `account::get_map_item`/`set_map_item`). ([0xmiden.github.io][4])
* **Note metadata, recipient restriction, and nullifiers** for permit notes, travel-rule envelopes, and replay-proof single-use tokens. ([0xmiden.github.io][5])
* **Transaction guards** like `tx::get_block_timestamp` and `tx::update_expiration_block_delta` for freshness/expiry. ([0xmiden.github.io][2])
* **Node RPC** endpoints including **CheckNullifiers** and **SyncNotes** to support clients that sync tags or verify spent permits. ([0xmiden.github.io][6])

---

## Architecture

### 1) Compliance Registry (public account)

**Storage layout (public):**

* `ALLOW_ROOT` (slot 1) — Merkle root of credential commitments (KYC-cleared).
* `REVOKE_ROOT` (slot 2) — sparse-tree root for revocations (optional).
* `EPOCH` (slot 3) — monotonically increasing “freshness” tag.
* `CP_KEYSET_ROOT` (slot 4, map optional) — Merkle root of approved Compliance-Provider keys.
* Optional **jurisdictional roots** in a **map** (slot 5) keyed by region code (`u32` → root). Use `set_map_item/get_map_item`. ([0xmiden.github.io][4])

**Public read procedure (registry):** export a read-only proc (no storage writes) returning the current roots & epoch. Your stablecoin will FPI-call it during `transfer`. FPI is explicitly for **read-only cross-contract calls**. ([0xmiden.github.io][3])

### 2) User credential commitment (offchain)

KYC provider issues a commitment

$$C = \mathrm{RPO256}\big(\mathrm{bind}(\text{senderpk}) \parallel \text{attrshash} \parallel \text{salt}\big)$$

plus a Merkle path $\pi$ into `ALLOW_ROOT`. Keep $C,\pi$ **local**; provide them as private advice to the prover. RPO is first-class in Miden’s stdlib for efficient hashing; MASM has memory-hash helpers. ([0xmiden.github.io][7])

> Binding to `sender_pk` (or account ID) prevents credential sharing. Only a holder of that key can use the cred.

### 3) Stablecoin Account (MASM) — the transfer gate

On `transfer`, the account code (running in the correct **contexts**—read at “Any”; write/mint/burn only in **Native & Account / Faucet**) does:

1. **FPI** to the Registry ⇒ fetch `ALLOW_ROOT`, `REVOKE_ROOT`, `EPOCH` (and CP keyset or regional root). FPI invocation surface is `tx::execute_foreign_procedure`. ([0xmiden.github.io][2])
2. **Verify membership**: recompute `ALLOW_ROOT` from $C,\pi$.
3. **(Optional) Verify absence** in `REVOKE_ROOT` (sparse-Merkle absence proof).
4. **Freshness**: require `registry_epoch ≥ min_epoch` (policy input). Use `tx::get_block_timestamp`/`get_block_number` and/or rotate `EPOCH` on the registry; caller asserts the minimum epoch. ([0xmiden.github.io][2])
5. **Proceed with spend**: move fungible asset from sender’s vault and create an output note (P2ID or custom) via `tx::create_note` + `tx::add_asset_to_note` (or wallet helpers), all in **Native & Account**. ([0xmiden.github.io][2])

### 4) Optional “TransferPermit” note (risk-based controls)

For corridors (>\$X, high-risk geos), require a one-time **Permit** note:

* **Mint** Permit from a **Compliance** account (public).
* **Bind** to `{sender_id_hash, receiver_id_hash, amount_range, expiry}` in **note inputs/metadata**.
* Enforce in a **note script**:

  * Check the Permit’s **sender** is the Compliance account (origin check via `note::get_sender`).
  * Check **expiry** using `tx::get_block_timestamp`/`get_block_number`.
  * Burn on spend (nullifier guarantees single-use). ([0xmiden.github.io][5])

### 5) Travel-rule envelope (optional)

Attach an encrypted IVMS-101 payload in the output note **metadata**; the chain need only enforce “envelope present,” keeping proofs small. Note metadata is a first-class field; recipient restriction (RECIPIENT) protects who can consume. ([0xmiden.github.io][5])

### 6) Issuer controls (freezes/recalls)

Use **P2IDR (reclaimable)** note paths or a time-boxed emergency branch gated by an issuer multisig. These are standard script patterns in Miden’s note model. ([0xmiden.github.io][1])

---

## MASM sketches (with Protocol Library calls)

### A) Registry: read-only public proc (returns roots & epoch)

*Context: Account (read-only body)*

```text
# export.read_roots => [ALLOW_ROOT, REVOKE_ROOT, EPOCH]
export.read_roots
    # slots 1,2,3 hold roots and epoch
    push.1  exec.account::get_item     # -> [ALLOW_ROOT]
    push.2  exec.account::get_item     # -> [REVOKE_ROOT, ALLOW_ROOT]
    swap    # order as you like
    push.3  exec.account::get_item     # -> [EPOCH, REVOKE_ROOT, ALLOW_ROOT]
    # leave 3 words on stack as outputs
end
```

We’ll reference this proc’s **root/hash** when invoking FPI. ([0xmiden.github.io][2])

### B) Stablecoin.transfer: fetch roots via FPI, verify, then transfer

*Context: mix of Any (reads) and Native & Account (state changes)*

```text
use.miden::tx
use.miden::account
use.std::crypto::hashes::rpo

# Public inputs (to verifier): [REGISTRY_ID_PREFIX, REGISTRY_ID_SUFFIX, READ_ROOTS_PROC_HASH, MIN_EPOCH]
# Private witness: [C, pi_allow..., (opt) pi_revoke_absent..., sender_pk, attrs_hash, salt]

export.transfer
    # 1) call foreign registry read proc: returns [ALLOW_ROOT, REVOKE_ROOT, EPOCH]
    #    stack must be [acc_id_prefix, acc_id_suffix, FOREIGN_PROC_ROOT, pad(n)]
    #    (push your constants/inputs accordingly)
    ... push.{READ_ROOTS_PROC_HASH} push.{REG_ID_SUFFIX} push.{REG_ID_PREFIX}
    exec.tx::execute_foreign_procedure
    # => [ALLOW_ROOT, REVOKE_ROOT, EPOCH]

    # 2) bind credential
    # stack: [..., ALLOW_ROOT, REVOKE_ROOT, EPOCH]
    # compute C' = H(sender_pk || attrs_hash || salt) and assert C == C'
    ...  # rpo::hash_* from stdlib
    # assert equality with witness C

    # 3) verify membership of C in ALLOW_ROOT (Merkle verify)
    ...  # merkle_verify(C, pi_allow, ALLOW_ROOT) using RPO building blocks

    # 4) optional non-membership in REVOKE_ROOT
    ...  # smt_verify_absent(C, pi_revoke_absent, REVOKE_ROOT)

    # 5) freshness
    push.{MIN_EPOCH} gt or eq    # EPOCH >= MIN_EPOCH
    assert

    # 6) perform spend: debit and create output note(s)
    # build recipient and note; add asset via tx::create_note + add_asset_to_note
    ...
end
```

**Where these calls come from:** `tx::execute_foreign_procedure`, `account::*`, `tx::create_note`, `tx::add_asset_to_note`, plus hashing helpers. Context constraints and I/O shapes are in the Protocol Library. ([0xmiden.github.io][2])

### C) Permit Note (script) — verify origin & expiry, then release

*Context: Note script*

```text
use.miden::note
use.miden::tx
use.std::sys

# Inputs (example): [SENDER_ID_HASH, RECEIVER_ID_HASH, MIN_AMOUNT, MAX_AMOUNT, EXPIRY_BLOCK]
begin
    # Ensure this permit was minted by the Compliance account
    exec.note::get_sender              # -> [sender_id_prefix, sender_id_suffix]
    # compare to known Compliance account ID (or verify via FPI against a registry of valid issuers)

    # Check expiry
    exec.tx::get_block_number          # -> [block_num]
    # assert block_num <= EXPIRY_BLOCK

    # (Optional) Recompute hashes to match sender/receiver/amount corridor passed in
    # If all checks pass, add assets to vault of consuming account
    call.miden::contracts::wallets::basic->wallet::receive_asset

    exec.sys::truncate_stack
end
```

Use **nullifiers** to ensure the permit is single-use; the node exposes **CheckNullifiers** to wallets/services. ([0xmiden.github.io][6])

---

## Policy toggles you can encode (cleanly)

* **One- vs two-sided checks:** require sender only, or both sender + receiver membership.
* **Freshness floor:** `epoch ≥ min_epoch`; rotate registry roots frequently. ([0xmiden.github.io][2])
* **Jurisdiction routing:** keep multiple `ALLOW_ROOT_*` in a map keyed by region; pick root based on a policy flag in stablecoin storage. Use `get_map_item/set_map_item`. ([0xmiden.github.io][4])
* **Issuer privacy:** commit to a set of **approved CP keys** as a Merkle root; prove the issuing key ∈ set without revealing which.
* **Emergency controls:** reclaimable path via P2IDR or time-boxed branch gated by issuer multisig in a note script. ([0xmiden.github.io][1])
* **Mempool staleness guard:** set a short transaction TTL via `tx::update_expiration_block_delta`. ([0xmiden.github.io][2])

---

## Minimal MVP checklist (with doc pointers)

**A. Deploy the Registry (public)**

* Define storage slots & (optionally) a **map** for regional roots (`set_map_item`, `get_map_item`). ([0xmiden.github.io][4])
* Implement `export.read_roots` (read-only). Consumers call it via **FPI**. ([0xmiden.github.io][3])

**B. Build the Stablecoin**

* Start from the fungible faucet/wallet templates; your transfer runs in **Native & Account**.
* Add compliance checks: FPI → roots/epoch, Merkle verifies, freshness, optional permit. Use `create_note`/`add_asset_to_note` to emit P2ID notes. ([0xmiden.github.io][2])

**C. Permit flow (optional)**

* Mint **Permit** notes from the Compliance account; in the permit script use `note::get_sender` (origin) & `tx::get_block_number` (expiry) before releasing assets to the consumer. ([0xmiden.github.io][5])

**D. Client UX**

* **Local proving** by default; **delegated proving** is supported if a user’s device is weak. ([0xmiden.github.io][1])
* Sync **tags** for relevant notes and verify spent permits via **CheckNullifiers** / **SyncNotes**. ([0xmiden.github.io][6])
* For public contracts, see “Interacting with Public Smart Contracts”; for FPI wiring, follow the FPI tutorial’s pattern of pushing the **foreign proc hash** and account ID prefix/suffix into `tx::execute_foreign_procedure`. ([0xmiden.github.io][8])

---

## Why this is “real” and not “mock”

* **Mock** gating relies on the relayer/wallet to block txs; users can bypass it.
* **Real** gating lives in **MASM** and is verified by the network: the ZK proof shows the transfer logic **FPI-read** the registry and enforced membership/freshness/permits; the operator only accepts valid proofs. ([0xmiden.github.io][2])

---

## Footnotes / gotchas

* Mind **contexts**: reads (Any), state writes (Native & Account), faucet ops (Faucet), auth-only nonces (Auth). The Protocol Library table is your source of truth. ([0xmiden.github.io][2])
* FPI is **read-only** by design; mutating the registry happens via a separate admin tx (registry’s native account). ([0xmiden.github.io][3])
* Note recipients & metadata: the **RECIPIENT** construction and tag semantics are documented; use them to restrict consumption and to let clients discover notes without over-revealing interest. ([0xmiden.github.io][5])
* Docs status: Miden’s docs and feature set are evolving (v0.8 at time of writing), so expect minor API shifts. ([0xmiden.github.io][1])

---

[1]: https://0xmiden.github.io/miden-docs/ "Introduction - The Miden book"
[2]: https://0xmiden.github.io/miden-docs/imported/miden-base/src/protocol_library.html "Miden Protocol Library - The Miden book"
[3]: https://0xmiden.github.io/miden-docs/imported/miden-tutorials/src/rust-client/foreign_procedure_invocation_tutorial.html "Foreign Procedure Invocation - The Miden book"
[4]: https://0xmiden.github.io/miden-docs/imported/miden-tutorials/src/rust-client/mappings_in_masm_how_to.html "How to Use Mappings in Miden Assembly - The Miden book"
[5]: https://0xmiden.github.io/miden-docs/imported/miden-base/src/note.html "Note - The Miden book"
[6]: https://0xmiden.github.io/miden-docs/imported/miden-node/src/user/rpc.html "Node gRPC Reference - The Miden book"
[7]: https://0xmiden.github.io/miden-docs/imported/miden-vm/src/user_docs/stdlib/crypto/hashes.html "std::crypto::hashes - The Miden book"
[8]: https://0xmiden.github.io/miden-docs/imported/miden-tutorials/src/rust-client/public_account_interaction_tutorial.html "Interacting with Public Smart Contracts - The Miden book"
