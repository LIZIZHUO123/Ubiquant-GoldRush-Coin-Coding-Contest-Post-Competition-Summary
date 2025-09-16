Ubiquant: GoldRush Coin Coding Contest

Post-Competition Summary

**Team Members**:

    Ningfeng Guo, Central University of Finance and Economics
    Yitong Zhang, Peking University
    Zizhuo Li, Southwestern University of Finance and Economics

# 1. Brief Recap of the Competition

* **Environment**: 17×17 discrete grid, 1v1, diagonal spawn. Five cell states: empty/coin/obstacle/bomb/player.
* **Dynamics**:

  * Coins drop each turn in the central 9×9 area; NPCs appear periodically (cycle Z), scatter coins along a path, then leave; bombs refresh periodically (cycle Y).
  * Each turn, each player has **3 single-step moves** (up, down, left, right). Out-of-bounds/obstacle/collision invalidates that action, remaining moves continue.
  * **Execution priority is decided by latency**; contested resources are granted to whoever arrives first; stepping on a bomb deducts X% of current coins, then the bomb disappears.
* **Objective**: After 300 turns, player with more coins wins.
* **Hard constraints**: Each turn must return actions within 1s; faster response yields advantage.

---

# 2. Solution to the Competition

## 2.1 Strategy Framework (Decision Flow + Key Formulas)

---

Overview: **Resolve conflicts first, advance steadily overall, divert to distant targets only if net gain is significant**. Priorities cascade from deterministic grabs to heuristic search to simulation-based gating. This avoids two common traps: chasing faraway coins inefficiently, and pointlessly colliding with the opponent over nearby coins.

---

Definitions:

* Current turn $t$, grid $G_t\in\mathbb{Z}^{17\times17}$ with cell values as defined.
* Our position $s_t=(x_t,y_t)$, opponent’s position $o_t$.
* Action set $\mathcal{A}=\{0,1,2,3\}$ for up/down/left/right with displacement $(DX[a],DY[a])$.
* Validity $V(p,a)=1$ if action $a$ from $p$ is legal.
* Bomb set $\mathcal{B}_t$, coin set $\mathcal{C}_t=\{u\mid G_t(u)>0\}$, cell coin value $v(u)=G_t(u)$.

Each turn outputs a 3-step action sequence $[a_1,a_2,a_3]$. Priority flow:


### Step 0: Endgame Short-Sighted Maximization (Hard Rule)

If $t\ge 295$, directly maximize **discounted 3-step payoff**:

$$
\max_{(a_1,a_2,a_3)\in \mathcal{A}^3}\quad R_{\text{3step}}(a_{1:3})
$$

where

$$
R_{\text{3step}}(a_{1:3})=\sum_{i=1}^{3}\gamma^{\,i-1}\cdot \big[v(p_i)\cdot \rho(p_i)\big] - \sum_{i=1}^{3}\mathbf{1}[p_i\in \mathcal{B}_t]\cdot L_i
$$

* $p_i$: position after step $i$; $\gamma\in(0,1)$.
* $\rho(p)$: **regional multiplier** (see Step 3).
* Li = ceil(X% * gold_{i-1}): loss if stepping on bomb (with holdings gold_{i-1}).  
  If no valid action, fallback: greedy 3 steps toward center.

---

### Step 1: Distances and Opponent Reachability (Common Base)

Use BFS to compute our shortest distances $d_p(u)$ and opponent’s distances $d_o(u)$, ignoring obstacles/invalid cells.
Also record **first-step sets** $\mathcal{F}(u)\subseteq\mathcal{A}$: all possible first moves from $s_t$ on shortest paths to $u$.

---

### Step 2: Conflict Detection and Local Grab

Define **3-step conflict set**:

$$
\mathcal{Q}=\{u\in \mathcal{C}_t \mid d_p(u)\le 3 \ \land\ d_o(u)\le 3\}
$$

If $\mathcal{Q}\neq\emptyset$, run **3-step local enumeration**:

$$
\max_{(a_1,a_2,a_3)\in \mathcal{A}^3}\quad R_{\text{3step}}(a_{1:3})
$$

Evaluate distinct coin captures within 3 steps, disallow duplicates.
Prune illegal moves; simulate opponent’s greedy 1-step response to avoid traps.
If $R_{\text{3step}}^\star>0$, adopt optimal $[a_1,a_2,a_3]$; else proceed to Step 3.

---

### Step 3: Gravitational Field Heuristic (Global Navigation)

For each coin $u\in\mathcal{C}_t$, distribute its value across its first-step set $d\in \mathcal{F}(u)$.
**Direction score**:

$$
S(d)=\sum_{u\in\mathcal{C}_t,\ d\in\mathcal{F}(u)}
\frac{v(u)}{|\mathcal{F}(u)|}\cdot \frac{1}{d_p(u)^2+\epsilon}\cdot W\big(\Delta(u)\big)\cdot \rho(u)
$$

where $\Delta(u)=d_o(u)-d_p(u)$. Opponent gap weight:

$$
W(\Delta)=
\begin{cases}
0.3,& \Delta<-6 \ \text{or}\ (\Delta\le -3 \land d_o(u)\le 3) \\
0.7,& 0\le \Delta\le 3 \\
1.3\cdot 0.9^{\max(0,d_o(u))},& 3<\Delta\le 6 \\
1,& \text{otherwise}
\end{cases}
$$

**Regional multiplier** $\rho(u)$ based on central vs. outer density:

$$
\rho(u)=
\begin{cases}
1+\alpha\cdot \max(D_{\text{ctr}}-D_{\text{out}},0), & u \in 9\times 9 \text{ center}\\
1, & \text{otherwise}
\end{cases}
$$

Densities from net new coins per area per step. $\alpha$=0.1.

Pick

$$
a^\star=\arg\max_{d\in\mathcal{A}} S(d)
$$

Break ties randomly to avoid predictable blocking.
Apply $a^\star$ in a rolling 3-step manner, updating after each simulated step.

---

### Step 4: Forward Evaluation and Net Gain Gating (High-Value Only)

To avoid chasing distant mirages, evaluate top-K coins:

1. Candidate set $\mathcal{U}_K\subset\mathcal{C}_t$: top $K$ by $v(u)$, with $K\le 100$.
2. For each $u$, find path $pi(u)$ minimizing **resistance cost**:

$$
C(\pi)=\sum_{e\in\pi}c(e),\quad
c(e)=
\begin{cases}
2,& \text{empty cell}\\
1/v(u')^2,& \text{coin cell }u'\\
+\infty,& \text{obstacle/opponent/bomb}
\end{cases}
$$

3. Simulate $H=18$ steps with turn-based play (opponent first, using heuristic). Track Gain_plan(u) and Gain_base
4. Net gain gate:

$$
\Delta(u)=\text{Gain}_{\text{plan}}(u)-\text{Gain}_{\text{base}}
$$

If $\max_{u\in\mathcal{U}_K}\Delta(u)\ge \tau$ (threshold $\tau=5$), adopt path to $u^\star$; else stick to baseline.

---

### Step 5: Bomb Risk and Threshold Avoidance

Current holdings $g$, bomb threshold $\theta_b$.

* If $g>\theta_b$, reject any move onto bomb.
* If $g\le \theta_b$, allow only if **expected net gain non-negative**:

Expected future gain − ⌈X% · g⌉ ≥ 0

Future gain approximated by short-horizon simulation.

---

### Step 6: Exceptions and Fallbacks

* Missing state info: default to “3 steps toward center.”
* If no legal moves two steps in a row: choose nearest safe direction prioritizing obstacle/bomb/opponent avoidance.

---

## Complexity and Latency Control (1s Guarantee)

* BFS: $O(17^2)$.
* 3-step enumeration: constant (≤64 paths) with pruning.
* Forward simulation: $K\le 100,\ H=18$ feasible <1s in C++.
* Data structures: arrays/deques, local grid copies.

---

## Key Formula Cheat Sheet

1. Direction score:

$$
S(d)=\sum_{u}\frac{v(u)}{|\mathcal{F}(u)|}\cdot\frac{1}{d_p(u)^2+\epsilon}\cdot W(d_o(u)-d_p(u))\cdot \rho(u)
$$

2. Regional multiplier:

$$
\rho(u)=
\begin{cases}
1+\alpha \cdot \max(D_{\text{ctr}}-D_{\text{out}},0), & u\in \text{center}\\
1,& \text{otherwise}
\end{cases}
$$

3. Three-step reward:

$$
R_{\text{3step}}=\sum_{i=1}^{3}\gamma^{i-1}\,v(p_i)\rho(p_i)-\sum_{i=1}^{3}\mathbf{1}[p_i\in\mathcal{B}_t]\cdot \lceil X\%\cdot \text{gold}_{i-1}\rceil
$$

4. Net gain gating:

$$
\Delta(u)=\text{Gain}_{\text{plan}}(u)-\text{Gain}_{\text{base}},\quad \max_u \Delta(u)\ge \tau
$$

5. Bomb net condition: Expected future gain − ⌈X% · g⌉ ≥ 0

---

## 2.2 Engineering and Latency Optimization

* **BFS reuse**: compute once, reuse across steps.
* **Pruning**: deduplicate in 3-step enumeration; conflict detection first.
* **Structures**: arrays/deques over heavy containers; branch ordering by hit rate.
* **Fallbacks**: missing info or blockages → safe center path, ensuring <1s response.

---

## 2.3 Strengths, Weaknesses, and Review

* **Strengths**:

  * Low latency and stable response, exploiting “first-move” advantage.
  * Quick reaction to nearby high-value coins, high win rate in conflict zones.
  * Explicit endgame handling to lock in final gains.

* **Weaknesses**:

  * No explicit **opponent modeling**, slow to adapt to blocking/ambush styles.
  * Center-bias multiplier is heuristic, may lag in abrupt shifts.

* **Improvements**:

  * Introduce **meta-strategy selector** (gravity vs. 3-step brute vs. block search) with multi-armed bandit or Bayesian switching.
  * Lightweight opponent profiling (recent trajectory + blocking tendency score) to adapt `w(·)`.

---

# 3. Links to Financial Markets and Quant Research Insights

Treat the game as a miniature “market microstructure sandbox.” Mappings:

## 3.1 Financial Market Analogies

* **Coins** ≈ **extractable liquidity/alpha**: dynamic distribution; center like liquid tickers or main trading hours.
* **Bombs** ≈ **adverse liquidity/impact costs**: gaps, hidden cancels, fees, slippage; larger holdings magnify losses.
* **Execution priority by latency** ≈ **queue/low-latency execution**: faster response means better fill probability, lower risk.
* **Obstacles and opponent** ≈ **market constraints and competitor strategies**: price limits, risk boundaries, blocking flows.
* **NPC coin drops** ≈ **periodic exogenous order flow**: event-driven liquidity injection.

---

## 3.2 Quantitative Research Lessons

**1. “Signal” ≠ “Profit”**

* Proximity to coins must account for **arrival time, interception risk, bomb costs**. Equivalent to modeling **realizable alpha** with impact, queue, and fill probability.
* Formal analogy:

$$
\text{Expected PnL}=\sum\_i \alpha_i\cdot \Pr(\text{fill}_i)-C_{\text{impact}}-C_{\text{fee}}
$$

Our weights `w(o_dist-p_dist)`,`1/p_dist^2`, and bomb thresholds are grid analogs.

**2. Latency is a Hidden Alpha Tax**

* Execution priority mirrors **matching delay and queue time**.
* In intraday alpha research, both **opponent behavior** (e.g., order book changes) and **execution time** matter.
* Reducing delay improves fill rates and reduces slippage; speed is not luxury, but profit-critical.
* Backtests must incorporate **fill probability, queue estimation, impact curves**, focusing on realized PnL.

**3. Risk Budget and Inventory Management**

* Bomb loss is multiplicative risk: more holdings → higher vulnerability, like inventory risk.
* Real markets similarly scale risk controls: reduce exposure, widen spreads, or hedge when inventory is high.
* Thresholded position management mirrors our bomb threshold logic.

---

**Conclusion**: The essence of this problem is extracting “realizable excess return” under **latency, congestion, and impact**. Strategies must jointly optimize signal quality, execution priority, and risk costs.


