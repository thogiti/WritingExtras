
#### Note
This is a mathematically formalized document of the proposal from Fikumni. You can read his original proposal [here.](https://github.com/fikunmi-ap/solana-improvement-documents/blob/restrict-tfm-bidspace/restrict-tfm-bidspace.md)

#### Overview
The proposal introduces a recommended priority fee system based on recent fee data and account contention. The goal is to improve fee predictability and block utilization while minimizing protocol changes. The system relies on a cache maintained by RPC nodes to map accounts to recommended fees.

---

### Key Components

#### Definitions and Notations
- Let $\mathcal{A}$ be the set of all accounts in the network.
- Let $\mathcal{T}$ be the set of all transactions in a block.
- Let $b_n$ denote the most recent block seen by an RPC node.
- Let $b_{n-1}, b_{n-2}$ denote the two previous blocks.
- Let $f_t$ be the fee paid by transaction $t \in \mathcal{T}$.
- Let $f_r(a)$ be the recommended priority fee for account $a \in \mathcal{A}$ in block $b_{n+1}$.
- Let $f_h(a)$ be the recommended high-priority fee for account $a \in \mathcal{A}$ in block $b_{n+1}$.
- Let $\text{contention}(a)$ be the number of transactions writing to account $a$ in a block.

---

### Cache Structure
The cache is an in-memory data structure maintained by RPC nodes. It maps accounts to a fee_data struct, which contains:
- The recommended fee $f_r(a)$ for block $b_{n+1}$.
- The median fee $\text{median}(a)$ paid for account $a$ in the last three blocks $b_n, b_{n-1}, b_{n-2}$.
- The p90 fee $\text{p90}(a)$ paid for account $a$ in the last three blocks $b_n, b_{n-1}, b_{n-2}$.

Mathematically, the cache $\mathcal{C}$ can be represented as:

$$C = \{ (a, f_r(a), \text{median}(a), \text{p90}(a)) \mid a \in A \}.$$

---

### Fee Recommendation Algorithm

#### Priority Fee Recommender
The priority fee recommender calculates the recommended fee $f_r(a)$ for account $a$ based on recent fee data and contention. The algorithm is as follows:

- Input: Account $a$, median fee $\text{median}(a)$, p90 fee $\text{p90}(a)$, contention $\text{contention}(a)$.
- Output: Recommended fee $f_r(a)$ for block $b_{n+1}$.

The recommended fee $f_r(a)$ is calculated using an exponential controller:

$$f_r(a) = \text{median}(a) \cdot \exp\left(\beta \cdot \left(\frac{\text{contention}(a)}{\text{contention}_{\text{max}}} - 1\right)\right),$$

where:
- $\beta$ is a sensitivity parameter (e.g., $\beta = 1$).
- $\text{contention}_{\text{max}}$ is the maximum contention observed in the network.

#### High-Priority Fee Recommender
The high-priority fee recommender calculates the recommended high-priority fee $f_h(a)$ for account $a$ based on the p90 fee:

$$f_h(a) = \text{p90}(a).$$

---

### RPC Processing

#### Processing Requests
When an RPC node receives a `getPriorityFee` request for a list of accounts $\mathcal{A}_{\text{req}} \subseteq \mathcal{A}$, it:
- Checks the cache $\mathcal{C}$ for all accounts $a \in \mathcal{A}_{\text{req}}$.
- If none of the accounts are in the cache, it returns the global median fee:

  $$f_{\text{global}} = \text{median}(\mathcal{T}).$$
  
- If one or more accounts are in the cache, it returns the recommended fee for the most contentious account:

  $$f_{\text{rec}} = \max_{a \in \mathcal{A}_{\text{req}}} f_r(a).$$

#### Processing Blocks
When an RPC node receives a new block $b_n$, it:
- Calculates the global median fee $\text{median}(\mathcal{T})$ for block $b_n$.
- Calculates the per-account median fee $\text{median}(a)$ and p90 fee $\text{p90}(a)$ for each account $a \in \mathcal{A}$ in block $b_n$.
- Updates the cache $\mathcal{C}$ with the recommended fees $f_r(a)$ and $f_h(a)$ for block $b_{n+1}$.

---

### Cache Eviction Policy
The cache $\mathcal{C}$ evicts accounts based on the following rules:
- Age-Based Eviction: Accounts with no transactions in the last three blocks are evicted:

  $$\text{evict}(a) \quad \text{if} \quad \text{most recent entry for } a \text{ is from } b_i \text{ where } n - i > 3.$$
  
- Fee-Based Eviction: Accounts with recommended fees $f_r(a) < f_{\text{global}}$ are evicted:

   $$\text{evict}(a) \quad \text{if} \quad f_r(a) < f_{\text{global}}.$$

---

### Mathematical Representation of the System

#### User Utility
The utility $u_t(b_t)$ of a user submitting transaction $t$ with bid $b_t$ is:

$$u_t(b_t) = \begin{cases}
v_t - b_t & \text{if } t \text{ is included in the block}, \\
0 & \text{otherwise},
\end{cases}$$

where $v_t$ is the user’s valuation for inclusion.

#### Block Construction
The block $B$ is constructed by selecting transactions $t \in \mathcal{T}$ that maximize the total fee revenue:

$$\max \sum_{t \in B} b_t \quad \text{subject to} \quad \sum_{t \in B} s_t \leq k,$$

where $s_t$ is the size of transaction $t$, and $k$ is the block size.

---

### Advantages and Limitations

#### Advantages:
- Improved Predictability: The recommended fees $f_r(a)$ and $f_h(a)$ provide users with a reference point for bidding.
- Maximal Block Utilization: The system allows blocks to be maximally packed, ensuring efficient use of block space.
- Minimal Protocol Changes: The system is implemented at the RPC level, avoiding invasive changes to the core protocol.

