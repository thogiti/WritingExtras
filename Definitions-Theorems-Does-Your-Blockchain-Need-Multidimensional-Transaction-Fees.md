
# Definitions and Theorems - Does Your Blockchain Need Multidimensional Transaction Fees?
**Nir Lavee, Noam Nisan, Mallesh Pai, Max Resnick**
https://arxiv.org/abs/2504.15438

## Definitions and Theorems

### Definition 1: Gas Measure
**Mathematical Statement:** A gas measure for a given set of primitive operations is a non-negative vector $g=(g_i)$. The gas measure is said to represent an operation-resource matrix $W=(w_{ij})$ with resource capacities $B=(B_j)$ if for any non-negative vector $x=(x_i)$:
$$\sum_i x_i g_i \le 1 \quad\implies\quad \forall\ j:\sum_i x_i w_{ij} \le B_j$$

**Explanation:** This defines a "gas measure" as a single set of costs ($g$) for blockchain operations. If a block's total gas cost is within a set limit (e.g., $\le 1$), it guarantees that all underlying, individual resource limits are also respected.

### Observation 1: Optimal Gas Measure
**Mathematical Statement:** For every operation-resource matrix $W=(w_{ij})$ with resource capacities $B=(B_{j})$, the gas measure given by $g_{i}=max_{j}w_{ij}/B_{j}$ represents W with B. Furthermore, g is minimal in the sense that for any gas measure g' that represents W with B we have that for all i, $g_{i}^{\prime}\ge g_{i}$.

**Explanation:** This provides the formula for the most efficient single-dimensional gas measure. The cost for an operation is set by its highest fractional usage of any single resource.

### Definition 2: $\alpha$-approximate representation
**Mathematical Statement:** A gas measure $g$ that represents an operation-resource matrix $W=(w_{ij})$ with resource capacities $B=(B_{j})$ is called an $\alpha$-approximate representation $(\alpha\ge1)$, if for any non-negative vector $x=(x_{i})$ we have that:
$$\Bigl(\forall j: \sum_{i}x_{i}\,w_{ij}\le B_{j}\Bigr) \implies \sum_{i}x_{i}\,g_{i}\le\alpha$$

**Explanation:** This definition quantifies the inefficiency of using a single gas measure. The factor $\alpha$ represents the maximum possible "overestimation" of resource usage by the single gas rule compared to the true, multi-dimensional constraints.

### Theorem 1: Single-Dimensional Approximability
**Mathematical Statement:** The single-dimensional approximability of an operation-resource matrix $W=(w_{ij})$ with resource capacities $B=(B_{j})$ is exactly the reciprocal of the value of the zero-sum game with utilities $u_{ij}=w_{ij}/(B_{j}\cdot g_{i})$ and where $g_{i}=max_{j}w_{ij}/B_{j}$ is the gas measure achieving this approximation (where the row player, who chooses i, is the minimizer.).

**Explanation:** This theorem provides a precise method to calculate the worst-case capacity loss ($\alpha$) from using a single gas measure. This loss is found by solving a specific zero-sum game based on the system's operations and resource costs.

***

## Extending to k-Dimensional Fees

### Definition 3: k-Dimensional Gas Measure
**Mathematical Statement:** A $k$-dimensional gas measure $A \in \mathbb{R}^{|I|\times k}$  $\textbf{represents}$ $W'$ if for any $x \ge 0$, we have:
$\forall r \leq q : \sum_i x_i A_{ir} \le 1 \;\implies\; \sum_i x_i w'_{ij} \le 1 \forall j.$


**Explanation:** This generalizes the gas measure concept from one to *k* dimensions. It defines a simplified set of *k* fee dimensions ($A$) to approximate a much larger, more complex set of *n* underlying resource constraints ($W^{\prime}$).

### Theorem 2: NP-Completeness of Optimal Partitioning
**Mathematical Statement:** The decision problem for optimal k-partitioning for gas measures (with $k=2$) is NP-complete.

**Explanation:** This theorem states that finding the best way to group many resource constraints into a smaller number of fee categories (even just two) is computationally very difficult. This is shown by reducing the problem from the Equal Cardinality Partition problem.

### Theorem 3: Representation via Upper-Bounding Factorization
**Mathematical Statement:** Let $W' \in \mathbb{R}^{|I|\times n}$ be the normalized operation-resource matrix. Suppose there exist matrices $A \in \mathbb{R}^{|I|\times k}$ and $B \in \mathbb{R}^{k\times n}$ such that:

- $W_{ij}^{\prime}\le(AB)_{ij}$ for all i, j. (Element-wise inequality) 
- The L1 norm of columns of B is bounded: $||B_{.,j}|| = \sum_{r=1}^k B_{rj} \le 1 \quad\text{for all }j = 1,\dots,n. $

Then, the k-dimensional gas costs given by $A_{ir}$ (i.e., $g_{i}=A_{i,\cdot})$ represents $W^{\prime}$.


**Explanation:** This theorem provides a method using Non-negative Matrix Factorization (NMF) to find a valid k-dimensional gas measure ($A$). It works by finding two smaller matrices whose product, $AB$, creates a safe upper bound on the true resource costs $W^{\prime}$.

### Theorem 4: Approximability of the Factorization Approach
**Mathematical Statement:** Let A be a k-dimensional gas measure for $W^{\prime}$ calculated using the factorization approach of Theorem 3. For each $l\in[k]$, define the utility matrix $U^{(l)}$ as $U_{ij}^{(l)}=w_{ij}^{\prime}/A_{il}$. Then, the approximability is the reciprocal of the minimum game value among the games $\{l\in[k]|U^{(l)}\}$.

**Explanation:** This theorem explains how to calculate the worst-case capacity loss for a k-dimensional fee system created via the factorization method. The calculation is similar to Theorem 1 but is performed for each of the *k* dimensions to find the overall approximation factor.

### Corollary 5: Factorization vs. Partitioning
**Mathematical Statement:** For any number of dimensions k, there exists a k-dimensional gas measure A via the factorization approach of Theorem 3 that is at least as good as the optimal k-dimensional gas measure via the partition approach of Theorem 2.

**Explanation:** This corollary states that the matrix factorization method for creating k-dimensional fees is guaranteed to be at least as efficient as, or better than, the method of simply grouping resources into a partition.

***

## Appendix: Proof-Related Statements

### Definition 4: Equal Cardinality Partition (ECP)
**Mathematical Statement:** Given an integer k and a multiset S of 2k positive integers that sum to 2T, the ECP problem is to partition S into 2 subsets of cardinality k that each sum to T.

**Explanation:** This defines a known NP-complete problem from computer science, which is used as a basis to prove that the problem of optimally partitioning blockchain resources is also NP-complete.

### Claim 6
**Mathematical Statement:** The instance of ECP has a solution if and only if the instance of the 2-gas-partition problem has a solution of approximability $k+T\epsilon$.

**Explanation:** This claim is the core of the proof for Theorem 2. It establishes a direct equivalence between solving an instance of the ECP problem and solving an instance of the resource partitioning problem for k=2.
