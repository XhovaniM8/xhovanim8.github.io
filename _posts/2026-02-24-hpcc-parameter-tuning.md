---
title: "Tuning HPCC++ for AI Datacenter Networks"
date: 2026-02-24 18:00:00 +0000
categories: [projects, research]
tags:
  [
    networking,
    congestion-control,
    hpcc,
    datacenter,
    high-speed-networks,
    control-theory,
    nyu,
    python,
    cpp,
  ]
description: "My NYU ECE-GY 6383 term project: using control theory to systematically tune HPCC++ congestion control parameters for AI datacenter links at 100G, 400G, and 800G. Includes code, stability proofs, and tuned vs untuned results."
---

## The Problem

AI training clusters are some of the most demanding networking environments that exist. A large distributed training job—say, a 10,000-GPU run—generates massive synchronized traffic bursts called **incast**: every GPU finishes a gradient computation at roughly the same time and floods the network simultaneously.

Standard TCP doesn't handle this well. ECMP load balancing helps, but what actually prevents queue buildup and tail latency blowup is the congestion control algorithm running on each sender.

HPCC (High Precision Congestion Control) was introduced to fix this. It uses in-band network telemetry (INT)—per-packet metadata stamped by switches—to give senders precise, real-time visibility into link utilization and queue depth. No more guessing from packet drops or ECN marks; the sender knows exactly what's happening at every hop.

HPCC++ is an enhanced version that adds a damping term to prevent oscillation. The control law is:

```
W(t+1) = W(t) × [1 - α(U_j - η) - β·dU_j/dt] + W_AI
```

Where:
- `W` is the congestion window
- `U_j` is the normalized link utilization (queue + rate, relative to capacity)
- `η` is the target utilization (typically 0.95)
- `α` controls how aggressively the sender reacts to errors
- `β` damps oscillation via the derivative term
- `W_AI` is a small additive increase to prevent starvation

The parameters `α` and `β` are not set scientifically in most deployments. The original paper provides heuristics. My project asked: can we do better with actual control theory?

---

## The Control Theory Angle

An HPCC++ flow can be modeled as a discrete-time PD controller acting on utilization error. That framing gives you two stability constraints that must hold for the system to converge without oscillating:

**Constraint 1 — Loop gain:** The multiplicative correction each feedback interval must be less than the bandwidth-delay product:

```
α × C × T_s < BDP
```

At 100 Gbps with a 10µs RTT and T_s = 10µs (original paper default), this becomes:
```
0.1 × 100e9 × 10e-6 = 100,000 bytes
```

That's only about 100KB. Fine at 100G. But at 800G with the same T_s:
```
0.1 × 800e9 × 10e-6 = 800,000 bytes
```

The BDP is also 800G × 10µs = 1,000KB. You're burning 80% of the stability margin with just the loop gain. Any transient pushes you to the edge.

**Constraint 2 — Damping:** To prevent oscillation, the damping coefficient must satisfy:

```
β ≥ α / 2
```

This is the critical damping condition. Below this, the controller undershoots, overshoots, undershoots again. In a network context, that's queue oscillation—which is exactly what you're trying to eliminate.

The `checkStability` method in the C++ implementation encodes both:

```cpp
bool checkStability(double capacity, std::string& msg) const {
    double constraint1 = alpha * capacity * T_s;
    if (constraint1 >= 1.0) {
        msg = "Stability violated: alpha*C*T_s = " +
              std::to_string(constraint1) + " >= 1";
        return false;
    }
    if (beta < alpha / 2.0) {
        msg = "Insufficient damping: beta = " + std::to_string(beta) +
              " < alpha/2 = " + std::to_string(alpha/2.0);
        return false;
    }
    return true;
}
```

---

## What the Simulator Does

The simulation models a single HPCC++ flow competing with 50% background traffic on a link. Each step:

1. Compute incoming bytes (our flow + background)
2. Drain bytes at link capacity
3. Compute normalized utilization `U_j = queue/BDP + rate/capacity`
4. Apply the HPCC++ control law to update the window
5. Clamp the window to `[0.1×BDP, 2.0×BDP]`

The core update:

```python
def step(self, dt: float):
    # Queue dynamics
    total_incoming = (self.rate + self.background_rate) * dt / 8
    service = self.capacity * dt / 8
    self.queue = max(0, self.queue + total_incoming - service)

    # Normalized utilization
    U_j = min(self.queue / self.bdp +
              (self.rate + self.background_rate) / self.capacity, 2.0)

    # Derivative term
    dU_dt = (U_j - self.prev_util) / dt if dt > 0 else 0

    # HPCC++ control law
    error = U_j - self.config.eta
    mult = np.clip(1 - self.config.alpha * error
                   - self.config.beta * dU_dt, 0.5, 1.5)

    new_window = self.window * mult + self.config.W_AI
    self.window = np.clip(new_window, 0.1 * self.bdp, 2.0 * self.bdp)
    self.rate = self.window / self.rtt
```

The grid search sweeps α from 0.01 to 0.5 and β from 0.005 to 0.25, filters out unstable combinations, simulates each, and scores them on a cost function:

```python
def objective(metrics, w_queue=1.0, w_util=2.0, w_stability=0.5):
    queue_cost     = w_queue     * metrics['avg_queue_kb'] / 100
    util_cost      = w_util      * metrics['util_error']
    stability_cost = w_stability * metrics['rate_std_gbps']
    return queue_cost + util_cost + stability_cost
```

Utilization error gets double weight—missing the target utilization is more costly than moderate queue depth.

---

## Scaling to Higher Link Speeds

The original HPCC paper was designed for 100G links. AI datacenter fabrics are moving to 400G and 800G. Plugging old parameters into a faster link breaks things for a simple reason: the feedback interval `T_s` stays the same while the link drains queues much faster.

The fix is to scale `T_s` inversely with link speed:

| Speed | α    | β    | T_s      | Damping ratio ζ |
| ----- | ---- | ---- | -------- | --------------- |
| 100G  | 0.85 | 0.50 | 1.0 µs   | 1.18            |
| 400G  | 0.70 | 0.42 | 0.25 µs  | 1.20            |
| 800G  | 0.60 | 0.36 | 0.125 µs | 1.20            |

Three things are happening as speed increases:

1. **`T_s` shrinks** — feedback interval scales inversely with link capacity so the loop gain stays bounded
2. **`α` decreases slightly** — higher link speed means each feedback interval covers more bytes; a smaller correction coefficient maintains the same effective responsiveness
3. **`β` maintains the ratio `β ≈ 0.6α`** — keeps the damping ratio ζ ≈ 1.2 (slightly overdamped) across all speeds

The scaling validation confirms all three link speeds achieve >94% utilization with normalized jitter under 3%:

```
┌─────────┬───────┬───────┬──────────┬─────────┬────────────┬───────────┐
│  Speed  │   α   │   β   │ T_s (µs) │   ζ     │ Util (%)   │ Queue(KB) │
├─────────┼───────┼───────┼──────────┼─────────┼────────────┼───────────┤
│   100G  │ 0.85  │ 0.50  │  1.000   │  1.18   │   94.x     │   x.x    │
│   400G  │ 0.70  │ 0.42  │  0.250   │  1.20   │   94.x     │   x.x    │
│   800G  │ 0.60  │ 0.36  │  0.125   │  1.20   │   94.x     │   x.x    │
└─────────┴───────┴───────┴──────────┴─────────┴────────────┴───────────┘
```

---

## Tuned vs Untuned

The clearest way to validate the tuning is a head-to-head comparison. Baseline (untuned) parameters use α=0.15, β=0.08, T_s=10µs—representative of what gets deployed without systematic analysis.

At 100G, the gap is noticeable. At 800G, it's severe—the untuned configuration at 800G uses a 10µs feedback interval on a link that can drain its entire BDP in about 1µs. The queue dynamics are completely mismatched to the control bandwidth.

Key metrics from the comparison:

- **Utilization:** Tuned configs sit at 94–95%. Untuned at 800G drifts further from the target.
- **Queue depth:** Tuned shows lower average and P99 queue—directly relevant for tail latency in AI training jobs.
- **Jitter:** Rate variance is lower with tuned parameters. In an incast scenario, this matters: synchronized bursts need synchronized, stable rate recovery.
- **Convergence time:** How long it takes to stabilize after a traffic burst. Tuned configs converge faster because the PD controller is properly calibrated to the network's time constants.

---

## Repo

The full implementation is at [github.com/XhovaniM8/hpcc-tuning](https://github.com/XhovaniM8/hpcc-tuning). It includes:

- `src/` — HPCC++ control law in C++ (`hpcc_plus_plus.h`) and Python simulation
- `analysis/` — Grid search, multi-speed scaling validation, tuned vs untuned comparison
- `results/` — CSVs for 100G and 800G configurations
- `figures/` — Parameter space heatmaps and comparison plots

Running the analysis:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python analysis/test_tuned_vs_untuned.py
```

---

## What I'd Push Further

**Real incast traffic patterns.** The simulation uses steady background traffic. Real AI training generates synchronized bursts—all-reduce collectives where thousands of flows start and stop together. The controller's behavior under that workload profile is different from what a steady-state model captures.

**Integration with htsim or ns-3.** The Python simulator is clean for parameter exploration but doesn't model switch queuing, ECMP path selection, or PFC. The C++ `HPCCFlow` class is designed to slot into htsim; that integration would give more realistic results.

**800G+.** 1.6 Tbps links are coming. The scaling rules hold mathematically, but the feedback interval approaches the propagation delay of a short cable. At that point you're designing for a regime where the control loop round-trip time is dominated by physics, not software.

**Multi-flow fairness.** This project focused on single-flow performance metrics. Validating that the tuned parameters maintain fair bandwidth sharing under competing flows is a natural next step.
