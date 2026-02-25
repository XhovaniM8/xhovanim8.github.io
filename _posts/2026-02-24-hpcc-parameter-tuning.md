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
    tail-latency,
    rdma,
  ]
description: "My NYU ECE-GY 6383 term project: using control theory and packet-level simulation to systematically tune HPCC++ congestion control for AI datacenter networks. Tail latency, parameter sweeps, and mechanism ablation."
---

## What This Is

For my ECE-GY 6383 (High-Speed Networks) term project at NYU, I treated HPCC++ congestion control as a feedback controller and asked: can you tune it systematically, and does it actually matter?

The short answer: yes, and specifically for **tail latency**—which is the metric that actually determines how fast a distributed AI training job runs.

Full paper and code: [github.com/XhovaniM8/hpcc-tuning](https://github.com/XhovaniM8/hpcc-tuning)

---

## Why Tail Latency Is the Real Problem

In distributed AI training, all-reduce and all-gather collectives are synchronized operations. Every GPU in a training job finishes a gradient computation at roughly the same time, then they all communicate before the next forward pass can start.

The step is only as fast as the **slowest flow**. If 999 flows finish in 1ms and one flow takes 10ms, the whole step takes 10ms.

This means optimizing mean flow completion time (FCT) is the wrong objective. What matters is p99 and max FCT—the tail. A congestion control algorithm that looks stable on average can still be a disaster for synchronized AI workloads if it occasionally creates stragglers.

At 100 Gbps with a 10µs RTT, the bandwidth-delay product is just 125KB. Even a few microseconds of unnecessary queue buildup from a misbehaving controller shows up in tail latency.

---

## HPCC++ in One Paragraph

HPCC++ uses In-band Network Telemetry (INT)—metadata stamped directly onto packets by switches—to give senders real-time visibility into queue length and transmission rate at every hop. No guessing from drops or ECN; the sender knows exactly what's happening.

The control law updates the congestion window with a PD-like rule:

```
W(t+1) = W(t) × [1 - α(U(t) - η) - β·ΔU(t)] + W_AI
```

Where `U(t)` is normalized utilization (queue + rate, both relative to capacity), `η` is the target utilization (0.95), `α` controls proportional responsiveness, and `β` damps rapid changes. `W_AI` is a small additive increment to prevent starvation.

Two implementation safeguards bound the update in practice:
- **Multiplicative clamping:** limits how large a single-step window change can be
- **Absolute cwnd bounding:** keeps the window within BDP-scaled limits

These safeguards are not incidental. They're central to how HPCC++ stays well-behaved in practice—and one of the main things the paper investigates.

---

## The Control Theory Framework

To organize the parameter space, I borrowed two concepts from control theory—but applied them carefully, because HPCC++ is not a clean linear system.

**Conservative stability heuristic:** Avoid changing in-flight data by more than O(BDP) per feedback interval. This gives:

```
α × C × T_s ≲ BDP = C × T
→ αmax ≈ T / T_s
```

This is intentionally conservative. It ignores the max-over-hops switching behavior (where the bottleneck link changes and causes nonlinearities), queue dynamics, and the effect of the safeguard clamps. But it's a useful organizing intuition: larger `T_s` → fewer control updates per RTT → smaller safe α.

**Damping proxy grouping:** To compare parameter configurations, I grouped `(α, β)` pairs by:

```
ζ = 2β / α
```

This is a proxy inspired by second-order damping intuition—not a true physical damping ratio. It's just a convenient way to normalize sweeps: for a fixed ζ, increasing α means proportionally increasing β to maintain similar "effective damping character." This made the sweep results interpretable without claiming more mathematical rigor than the model supports.

---

## Experiments: What Actually Ran

The main experiments used **csg-htsim**, a packet-level datacenter network simulator. The setup:

- Fat-tree topology, 432 nodes, 100 Gbps links
- ECMP routing (`-strat ecmp_host`)
- Sustained heavy workload (`heavy_384.txt`) — designed to stress the controller under persistent congestion
- Metrics: mean FCT, p99 FCT, max FCT, completion ratio, retransmissions
- Sweeps over `(α, β)` grouped by ζ, with all other parameters fixed

The Python implementation in the repo (shown below) is supplementary analysis for building intuition about the control law. The packet-level htsim experiments are the source of the paper's results.

The simulator was extended to expose HPCC++ parameters as command-line flags, enabling repeatable sweeps without code changes per run.

---

## What the Sweep Found

The tail behavior versus α is **non-monotonic**.

Too small: the controller converges slowly. Flows that start late or encounter persistent competition spend too long below target utilization. Tail latency suffers.

Too large: proportional control becomes aggressive enough to amplify feedback noise, max-over-hops switching, and transient queue spikes. Window oscillation increases. Occasional large swings in cwnd create bursts that disproportionately impact stragglers. Max FCT rises.

The sweet spot in our sweep: **moderate α with lower ζ groupings**. This region improves p99 and max FCT relative to default-like settings, primarily by reducing straggler completion time rather than improving the mean (which barely moves).

This is exactly the AI-training-relevant result. If you're optimizing a distributed training job, what you care about is the worst-case flow in each communication round, not the average.

---

## The Ablation Study

The more interesting finding came from the ablation. I tested four variants:

| Variant              | Description                              |
| -------------------- | ---------------------------------------- |
| `p_only`             | Proportional term only, no derivative    |
| `pd`                 | Full HPCC++ baseline                     |
| `pd_no_cwnd_clamp`   | PD control, absolute cwnd bound disabled |
| `pd_no_mult_clamp`   | PD control, multiplicative clamp disabled |

Disabling either safeguard degrades tail metrics. Disabling both makes things significantly worse.

This resolves an apparent paradox: HPCC++ appears tolerant to wide gain ranges in practice, yet the underlying control update is sensitive. The tolerance isn't a property of the PD law—it's a property of the bounded implementation. The clamps expand the operationally safe region by limiting how far any single bad update can move the window.

Practical implication: if you're deploying HPCC++ and thinking about stripping out the safeguards to simplify the implementation, don't. They're part of the control design, not dead code.

---

## The Python Simulation

Alongside the htsim experiments, the repo includes a Python simulator for building intuition about the control law under parameter variations. The core update matches the HPCC++ spec:

```python
def step(self, dt: float):
    # Queue dynamics with background traffic
    total_incoming = (self.rate + self.background_rate) * dt / 8
    self.queue = max(0, self.queue + total_incoming - self.capacity * dt / 8)

    # Normalized utilization
    U_j = min(self.queue / self.bdp +
              (self.rate + self.background_rate) / self.capacity, 2.0)

    # Derivative term for damping
    dU_dt = (U_j - self.prev_util) / dt if dt > 0 else 0

    # HPCC++ multiplicative update
    error = U_j - self.config.eta
    mult = np.clip(1 - self.config.alpha * error
                   - self.config.beta * dU_dt, 0.5, 1.5)

    new_window = self.window * mult + self.config.W_AI
    self.window = np.clip(new_window, 0.1 * self.bdp, 2.0 * self.bdp)
    self.rate = self.window / self.rtt
```

And the C++ header (`hpcc_plus_plus.h`) implements the same controller in a form designed to integrate with htsim or ns-3:

```cpp
// HPCC++ control law
double utilizationError = U_j - params_.eta;
double dampingTerm = params_.beta * dU_dt;
double multFactor = std::max(0.5, std::min(1.5,
    1.0 - params_.alpha * utilizationError - dampingTerm));

double newWindow = window_ * multFactor + params_.W_AI;
window_ = std::max(params_.minWindow, std::min(params_.maxWindow, newWindow));
rate_ = window_ / baseRTT_;
```

---

## Practical Takeaway for Operators

The paper suggests a conservative tuning workflow:

1. Start from a default-like baseline. Validate completion ratio and zero retransmissions under your target workload.
2. Increase α gradually. Watch p99 and max FCT—not mean. Stop before max FCT begins rising.
3. Adjust β proportionally via ζ groupings. Too little damping → overshoot. Too much → slow convergence.
4. Keep the safeguards on. Ablation results are unambiguous: removing clamps degrades tail behavior and reduces safety margin.

The deeper takeaway: HPCC++ is tunable, and tuning specifically for tail metrics under sustained load is achievable without algorithmic changes. The gap between a default configuration and a tuned one shows up in p99 and max FCT—exactly the numbers that determine AI training step time.

---

## Limitations

The experiments are specific to one topology, one buffer sizing, ECMP routing, and one traffic matrix. Other workload mixes—bursty incast, mixed RPC and collective traffic—may produce different sensitivity profiles. The simulation also depends on how accurately csg-htsim models INT feedback and queue dynamics under the actual line-rate behaviors of modern switches.

And zero retransmissions is a practical robustness indicator, not a formal stability proof.

---

*Code, sweep scripts, and plotting utilities: [github.com/XhovaniM8/hpcc-tuning](https://github.com/XhovaniM8/hpcc-tuning)*
