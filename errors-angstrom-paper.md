# Some errors and typos in the Angstrom paper
[Angstrom white paper](https://app.angstrom.xyz/whitepaper-v1.pdf): 

https://app.angstrom.xyz/whitepaper-v1.pdf

1. **Continuity claim for the net-supply curve.**

You state the bisection method works because $T_X(p)$ is “monotone and continuous” over the feasible interval. With indivisible exact-in/exact-out orders and activation thresholds, $T_X(p)$ is monotone non-decreasing but can have jump discontinuities at limit prices. Bisection can still be used to locate a root or a smallest price with non-negative excess, but the continuity claim should be removed and the algorithm described as bracketing the sign (or selecting the minimal $p$ with $T_X(p)\ge 0$).

2. **Minor variable mix-up in Theorem 3.2 proof.**
The proof says “the marginal price on the pool is given by the spot price $S_t$,” when the pool’s marginal price is $P_t$, and you’re showing $P_t$ equals $S_t$ in equilibrium. Suggest: “the marginal price on the pool equals the pool spot price $P_t$” … which in equilibrium equals $S_t$.

3. **“Less then” typo inside a theorem statement (but also affects clarity).**
Theorem 3.3’s proof uses “less **then** or equal,” which should be “less **than** or equal.” More importantly, it would help to restate the logic: fee-inclusive willingness to buy $\le S_t$,  sell $\ge S_t$ maps to fee-adjusted limits $S_t(1-\gamma)$ and $S_t/(1-\gamma)$; therefore $p_e$ lies in $[S_t(1-\gamma), S_t/(1-\gamma)]$.

4. **Naming inconsistency for digests.**
You define **LocalDigest** and **MergeDigest**, but later refer to **StateDigest** instead of LocalDigest (“Both **StateDigest** and MergeDigest…”), which is inconsistent and confusing. Please replace **StateDigest** with **LocalDigest**.

5. **Figure 2 caption: “MergedDigest” vs “MergeDigest.”**
Caption mixes **MergeDigest** and **MergedDigest**. Use **MergeDigest** consistently.

6. **Strong assumption: same-block inclusion** 
**Where:** “For the purposes of this section, we assume … no censorship from the Ethereum block proposer.”

It’s a strong assumption for liveness/optionality; readers may misinterpret it as guaranteed.
Add a short *“If inclusion slips a block …”* fallback paragraph (what happens to cancels, last-mover option bounds, and settlement semantics).

7. **Minor typos**

* **“Angstromnodes”** → “Angstrom nodes” 
* **“contractmaintains”** → “contract maintains”
* **“Witholding”** → “Withholding” 
* **“dobule spend”** → “double spend”
* **“Wewant”** → “We want” 

## Some Nice to Add Sections

1. **Censorship / missed-inclusion fallback.** Describe what happens if the canonical bundle doesn’t land in the same block (order validity, cancel guarantees, and any changes to bidder optionality).

2. **Builder diversity / revert-protection dependencies.** Name likely providers, describe failover, and state how you detect/penalize misbehavior in this layer (ties to the “top-of-block-only” backstop).

3. **Auction design sensitivity.** Briefly discuss bid shading and collusion risks in a first-price common-value auction and whether sealed-bid or commit-reveal tweaks were considered (you can still keep first-price; just acknowledge the tradeoffs).

4. **Empirics.** A short backtest on historical CEX–DEX basis with simulated validation delays would make the LP P\&L claims much more persuasive (even synthetic data would help).

