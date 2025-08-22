
# [Eclipse’s zkSVM - Bridging the ZK Gap on the SVM](https://www.eclipselabs.io/blogs/eclipses-zksvm-bridging-the-zk-gap)

https://www.eclipselabs.io/blogs/eclipses-zksvm-bridging-the-zk-gap


Eclipse just proved a big idea: you can verify Solana-style transactions with math, not trust. It’s called **zkSVM** (on **RISC Zero**). Transaction-level proofs work today; block-level and fraud proofs are next.

1. **What’s the point?**
   Blockchains should be fast *and* trustworthy. zkSVM lets us attach a compact mathematical “receipt” that a transaction followed the rules—so you rely less on operators and more on math.

2. **What’s the SVM?**
   Think of the **Solana Virtual Machine (SVM)** as the engine that runs programs on Solana-style chains. zkSVM is a way to *prove* that engine ran correctly without replaying everything yourself.

3. **Why Eclipse built this**
   EVM chains already enjoy lots of ZK options. SVM chains didn’t. Eclipse’s zkSVM closes that gap so Solana-style apps can get validity-grade assurances.

4. **How it works (simple)**
   We re-execute a transaction inside a secure sandbox (RISC Zero) and produce a proof (a tiny receipt) that says, “These inputs produced these outputs under the rules.” Anyone can check the receipt—fast.

5. **“But Solana is parallel!”**
   True. Solana groups non-conflicting transactions. Those can be proven **one-by-one** in a fixed order and you get **the same final outcome** as parallel execution. So parallelism isn’t a blocker.

6. **What’s live now**
   **Transaction-level proofs**: one transaction (or a conflict-free group) → one proof. No app changes required. Early costs/times are reasonable and trending down as the ZK stack improves.

7. **What’s missing for full blocks**
   We also need to prove accounts really belonged to the chain’s state. That’s what **AlDBaran** brings: per-account state proofs (multiproofs) so the math receipt ties back to the actual chain state.

8. **Near-term roadmap**
   (1) Plug in AlDBaran state proofs
   (2) Prove grouped transactions in parallel and aggregate receipts
   (3) **Block-level proofs**
   (4) **Single-step fraud proofs** so disputes can be settled on L1 quickly

9. **Why users should care**
   Stronger safety with less trust in middlemen. If something goes wrong, there’s a cryptographic record of what *should* have happened—easier to detect, challenge, or roll back bad behavior.

10. **Why builders should care**
    Verifiable execution unlocks new products (auditable trading, compliance-friendly apps) without re-writing for a new VM. You keep SVM performance while adding validity-style guarantees.

11. **Economics & performance**
    Proof costs/time are already acceptable for many cases and improve as RISC Zero/hardware advance. Batching and aggregation make it even cheaper at scale.

12. **Bottom line**
    No fundamental blockers left, just engineering. Eclipse’s zkSVM brings “math-verified” execution to the SVM today and paves the path to full block proofs and fast fraud resolution next.


