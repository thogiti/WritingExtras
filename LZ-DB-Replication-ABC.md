# Near-Real-Time, Order-Preserving Replication with Watermark Pulls - LayerZero Problem

## Problem Description

Data arrives into **A** every 30 seconds. Each record is replicated to **B** with an **SLA**: every record must reach **B** within **5 minutes**. We must then move data from **B → C**. System **C** has two modes:

* **Boot mode:** catch up as fast as possible (may violate “no poll failures”).
* **Live mode:** steady-state, near real-time, **no poll failures**, and **order-preserving** (“saved as they arrive in A”).

There’s **no timestamp in A**. We want a mathematically sound policy for when/how C pulls, when to switch to live, expected latency, failure recovery, SLA violations, and buffer sizing.

## Ground Rules

1. **No poll failures (live mode):** don’t pull if B has nothing ready.
2. **Near real time:** minimize end-to-end latency.
3. **Order-preserving:** C applies records in the order they arrived to A.
4. **Boot mode exception:** can ignore “no poll failures” to catch up fast.

---

## A Minimal Mathematical Model - Part A

* Arrivals into A every $T=30$ s. Index records by $n\in\mathbb{N}$; arrival times $a_n = nT$.
* A→B delay $D_n\ge 0$ with SLA $D_n\le S=300$ s (5 min). Arrival to B: $b_n=a_n+D_n$.
* Let $\mathcal{P}_B(t)=\{n: b_n\le t\}$ be records present in B by time $t$.

**Ready (watermark) prefix $r(t)$:** the largest contiguous index safe for C to pull.

* **Case $G=1$** (A→B preserves A’s order or B exposes a strict source sequence):

  $$r(t)=\max\{n:\{1,\dots,n\}\subseteq \mathcal{P}_B(t)\}.$$
* **Case $G=0$** (A→B may reorder and A exports no sequence/timestamp): enforce an **SLA holdback**

  $$r(t)=\Big\lfloor \tfrac{(t-S)^+}{T}\Big\rfloor,$$

  optionally clamped by actual contiguity to avoid over-advancing during SLA breaches.

Let $x(t)$ be C’s last applied index; **backlog** $B_C(t)=r(t)-x(t)$.
Ingress rate $\lambda=1/T=1/30$ rps. C’s apply rates: $\mu_{\text{boot}}$ (fast), $\mu_{\text{live}}$ (steady).

---

## Questions & Answers, Driven by Our Mathematical Model

### 1) How often should C pull from B?

**Event-driven on the watermark.** In live mode, pull only when $r(t)>x(t)$:

* Long-poll/notify on $r(t)$ → extra latency $W_{\text{poll}}\approx 0$.
* If polling, sample every $\tau$ seconds; adds at most $\tau$ (expected $\tau/2$).

**Result:** cadence is a latency knob, not correctness. Correctness comes from only pulling when $r>x$.

---

### 2) When to switch C from boot → live?

Switch after a short stability window (e.g., 5 min) once:

1. **Backlog small:** $B_C(t)=r(t)-x(t)\le K$, with $K=\lambda W$ for $W=2\text{–}4$ min.
2. **Capacity:** $\mu_{\text{live}}\ge \lambda(1+\varepsilon)$ for some margin $\varepsilon>0$.
3. **Replication stable:**

   * If $G=0$: observed $D_n$ respects the 5 min SLA so the holdback is valid.
   * If $G=1$: no gaps in the contiguous prefix.

---

### Minimum end-to-end latency (live mode)

For record $n$: $L_n = D_n + W_{\text{poll}} + W_{\text{queue}} + \delta_{\text{apply}}$.

* **If $G=1$** (ordered or sequenced): **floor ≈ seconds**: $L_n \gtrsim \inf D_n + \delta_{\text{apply}}$.
* **If $G=0$** (need SLA holdback $S$): **floor ≈ $S$**: $L_n \ge S+\delta_{\text{apply}}$ ≈ **5 min**.

---

### Recovery after failure/restart

Let downtime be $T_d$. Backlog grows by $\lambda T_d$. Boot-mode drain time:

$$
T_{\text{drain}}=\frac{\lambda T_d}{\mu_{\text{boot}}-\lambda}.
$$

Resume from durable offset $x_0$, re-enter live when $B_C\le K$. Use idempotent writes (or unique keys) at C.

---

### If A→B violates its 5-minute SLA

A breach means some $D_n>S$.

* Safe definition is always the **contiguous prefix present**:

  $$r(t)=\max\{n:\{1,\dots,n\}\subseteq \mathcal{P}_B(t)\}.$$
  
* If using time-based holdback, **clamp by contiguity** to avoid over-advancing during gaps.
* Operationally: $r(t)$ stalls; C naturally waits (no empty pulls). Alert; optionally offer a business-approved “best-effort” mode that relaxes ordering.

---

### How much buffer does B need?

B must cover **holdback + C downtime + jitter**.

Let $H=0$ if $G=1$, else $H=S$ (5 min); downtime $T_d$; jitter $J$.
Records:

$$N_B \approx \lambda \,(H+T_d+J).$$

Bytes: $N_B\times S_{\text{row}}$ (row+index size).

*Example:* $G=0$, $H=300$ s, $T_d=2$ h, $J=120$ s, $\lambda=1/30$ → $N_B\approx 252$ records.

---

## Implementation Cheat-Sheet

* Add a **monotonic `source_seq`** in B (preferably from A’s commit/LSN) and compute **`ready_seq`** $=r(t)$.
* **Live mode:** long-poll `ready_seq`; pull $(x, r]$ in order; persist offset.
* **Boot mode:** high-throughput batches until $B_C\le K$.
* If no `source_seq`/order: apply the **5-minute holdback** $r(t)=\lfloor (t-S)^+/T\rfloor$ **clamped by contiguity**.

**Key trade-off:** exporting a strict source order from A collapses latency from \~5 min to **seconds** while keeping all rules intact.

## Poisson Modeling - Part B
[Poisson](https://fiveable.me/stochastic-processes/unit-8/basic-queueing-models/study-guide/gKouMFxR7bFBtaK7) is a great *planning* model**, but our **correctness policy (watermark + order-preserving pulls)** shouldn’t depend on it. Here’s how to “Poissonize” the setup and what you gain.

### When Poisson makes sense

* If arrivals to **A** are truly periodic (every 30s), the exact model is **deterministic** (renewal with CV=0).
* In practice you often have jitter, bursts, or **many independent sources** superposed. By the **[Palm–Khintchine](https://en.wikipedia.org/wiki/Palm%E2%80%93Khintchine_theorem)** principle, superposing many renewal streams tends to a **Poisson** process. So Poisson is a good approximation for capacity/buffer/latency analysis.

### Poissonized model

Let $N_A(t)$ be a **Poisson($\lambda$)** counting process with $\lambda=\tfrac{1}{30}$ rps. Let interarrival times be i.i.d. $\text{Exp}(\lambda)$. Let replication delays $D_k\ge 0$ be i.i.d., independent of arrivals, with SLA target $D_k\le S=300$s (or at least $\Pr[D_k>S]$ very small).

* Arrival times into **A**: $\{A_k\}$.
* Arrival times into **B**: $B_k = A_k + D_k$.

**Mapping/displacement theorem:** shifting a Poisson process by i.i.d. delays yields **another Poisson process of the same rate**. Thus the arrival process into **B** is also Poisson($\lambda$) (though **order** may be scrambled by the shifts).

### Watermark (ready prefix)

We still need **order in C** (“saved as they arrive in A”). Without a source sequence from A, we enforce it via a **holdback**:

* If you *do* have a strict source sequence (or replication preserves order): C can apply as soon as data lands; the “ready” index $r(t)$ advances with the Poisson($\lambda$) arrivals to **B**.
* If not: choose a holdback $H$ (worst-case or high quantile of $D_k$). Then the largest index surely safe at time $t$ is

  $$r(t) \;=\; N_A(t - H).$$

  You don’t observe $N_A$ directly, but in steady state $N_A(t-H)$ and $N_B(t)$ have the same rate $\lambda$; operationally, you compute $r(t)$ as the **largest contiguous prefix present in B** and (if you want a time rule) clamp it by $t-H$.

> **Takeaway:** Poisson helps you quantify *how fast* $r(t)$ moves and how bursty it is; the **order-preserving policy** (apply only up to $r(t)$) remains the same.

### Latency with a Poisson arrival stream

Let $\mu$ be C’s sustained apply rate (records/s), service time $S_c=1/\mu$. The end-to-end latency for record $k$:

$$L_k \;=\; D_k \;+\; W_k \;+\; \delta_{\text{apply}},$$

where $W_k$ is queueing delay at C.

* For **M/G/1** (Poisson arrivals, general service), mean waiting time (Pollaczek–Khinchine):

  $$\mathbb{E}[W] \;=\; \frac{\lambda\,\mathbb{E}[S_c^2]}{2\,(1-\rho)},\quad\rho=\lambda/\mu.$$

  If C’s per-record work is near-deterministic (**M/D/1**), $\mathbb{E}[S_c^2]=1/\mu^2$, so

  $$\mathbb{E}[W] \approx \frac{\rho}{2\mu(1-\rho)}.$$
* **Kingman (G/G/1) approximation** for mean wait:

  $$\mathbb{E}[W] \;\approx\; \frac{\rho}{1-\rho}\cdot \frac{c_a^2 + c_s^2}{2}\cdot \frac{1}{\mu},$$

  with $c_a$ and $c_s$ the coefficients of variation of interarrival and service. For Poisson, $c_a=1$; for deterministic service, $c_s=0$.

**Minimum latency floor**

* If you have a source sequence (no holdback): $\min L \approx \min D + \delta_{\text{apply}}$ (seconds if replication is quick).
* If you rely on holdback $H$: $\min L \ge H + \delta_{\text{apply}}$ (≈ 5 minutes if $H=S$).

### Buffer sizing & downtime, with Poisson math

If C is down for $T_d$ seconds, the backlog created is

$$N \sim \text{Poisson}(\lambda T_d).$$

If you also hold back $H$ seconds for ordering, the “always-there” cushion is $\lambda H$. So design for (say) the 95th percentile:

$$N_{95} \;\approx\; \lambda (T_d + H) \;+\; 1.645\,\sqrt{\lambda (T_d + H)}.$$

Multiply by average bytes per record to size **B**.

### SLA violations under Poisson

If $\Pr[D_k>S]>0$, a time-based rule with $H=S$ can over-advance. The **safe** rule is still:

* set $r(t)$ to the **largest contiguous prefix actually present in B**, and
* optionally cap it by $t-H$ if you keep a time rule.
  C then long-polls $r(t)$; no empty pulls occur in live mode.

