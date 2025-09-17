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

## A Minimal Mathematical Model

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
