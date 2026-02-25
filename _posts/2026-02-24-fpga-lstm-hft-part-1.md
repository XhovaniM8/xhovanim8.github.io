---
title: "I Tried to Put an AI Inside Trading Hardware (Part 1: The Problem)"
date: 2026-02-24 10:00:00 +0000
categories: [projects, research]
tags:
  [
    fpga,
    lstm,
    high-frequency-trading,
    order-book,
    market-microstructure,
    deep-learning,
    nyu,
    capstone,
    quant,
    fintech,
  ]
description: "My NYU capstone project: building a real-time market behavior classifier using an LSTM neural network on FPGA hardware. Part 1 covers the problem, the finance context, and why regular hardware isn't fast enough."
---

## What This Is

For my Financial & Risk Engineering capstone at NYU Tandon, I built a prototype framework for classifying market behavior in real time using an LSTM neural network deployed on an FPGA.

The short version: markets move fast, bad actors exploit that speed, and if you can detect their patterns early enough—in microseconds—you might be able to do something about it. FPGAs are one of the few hardware platforms where "microseconds" is a realistic target.

This is Part 1. It covers the problem, the finance context, and why this is hard. Part 2 covers what I actually built, the results, and where it broke down.

---

## Why Speed Is Everything in HFT

In high-frequency trading, microseconds determine who wins.

You're not competing on insight. You're competing on reaction time. If your system is 10 microseconds slower than a competitor's, you will consistently trade at worse prices, fill orders after the market has moved, and systematically lose money to faster participants. This is called adverse selection.

Traditional CPU- and GPU-based architectures have a fundamental problem here: they're unpredictable. Operating system overhead, context switching, cache misses, network jitter—all of these introduce variability. You can optimize software down to low milliseconds. Getting consistently into the single-digit microsecond range, with deterministic timing, is a different problem entirely.

FPGAs solve this differently. Instead of running code on a general-purpose processor, you program the hardware itself. There's no OS. No scheduler. The logic is the circuit. The result is deterministic, pipelined processing that can operate at wire speed.

Prior work has demonstrated LSTM inference on FPGAs exceeding 13 GOP/s and latency reductions of over 250x compared to equivalent CPU implementations. That's the gap we're working in.

---

## The Limit Order Book

Before getting into the hardware, it helps to understand what we're actually analyzing.

At the center of every electronic market is the **limit order book (LOB)**—a real-time ledger of all outstanding buy and sell orders, organized by price and time priority. It updates continuously, on millisecond to microsecond timescales.

There are three standard levels of detail:

- **Level 1 (L1):** Best bid and best ask only. One-dimensional.
- **Level 2 (L2):** Multiple price levels on each side. You can see the "depth" of the market.
- **Level 3 (L3):** Individual orders at each price level, in queue order. Full visibility into who is waiting for what.

A snapshot of the L2 book looks something like this:

| Bid Level | Bid Price | Bid Volume | Ask Price | Ask Volume | Ask Level |
| --------- | --------- | ---------- | --------- | ---------- | --------- |
| 1         | 100.05    | 500        | 100.10    | 350        | 1         |
| 2         | 100.00    | 750        | 100.15    | 600        | 2         |
| 3         | 99.95     | 1200       | 100.20    | 450        | 3         |
| 4         | 99.90     | 850        | 100.25    | 300        | 4         |
| 5         | 99.85     | 600        | 100.30    | 550        | 5         |

The question my model tries to answer: given the last several snapshots of this book, which direction does the mid-price move in the next five steps?

---

## The Adversarial Problem

Standard indicators—moving averages, volume spikes, bid-ask spread—struggle in modern markets because many microstructure patterns are adversarial by design.

Three examples:

**Spoofing:** A large order appears on one side of the book, inducing other participants to react. The spoofer then cancels before execution. The apparent supply or demand was never real.

**Liquidity sweeps:** Aggressive orders rapidly consume multiple price levels on one side of the book. Often precedes a directional move.

**Momentum ignition:** Coordinated directional pressure that triggers other participants' momentum strategies, then reverses once enough people have piled in.

These patterns are subtle, transient, and designed to be hard to detect. Rule-based systems with static thresholds can't keep up. A model that learns from sequential order book data directly—and can run in hardware—is theoretically better suited.

---

## Adverse Selection Has a Price Tag

This isn't abstract. The costs are measurable.

Slower participants who trade against faster, better-informed counterparties typically incur 1–3 basis points of slippage per trade. For a desk executing $100M daily, that's $10,000–$30,000 per day—roughly $2.5–$7.5M annually.

Budish et al. estimated that latency arbitrage generates around $1 billion annually in U.S. equities alone. Tabb Group pegs each microsecond of latency reduction in HFT at approximately $100,000/year in trading advantage.

The financial incentive for hardware-accelerated pattern recognition is enormous.

---

## Why Not Just Use a GPU?

GPUs are great for training models. They're not great for this specific deployment problem.

| Platform     | Benefits                                   | Drawbacks                                     |
| ------------ | ------------------------------------------ | --------------------------------------------- |
| FPGA         | Low latency, deterministic, reconfigurable | Complex dev cycle, resource constraints       |
| ASIC         | Lowest latency, power-efficient            | Very high cost, zero reconfigurability        |
| CPU          | Easy to program, widely available          | Higher latency, limited parallelism for MATMUL |
| GPU          | Good for parallel/matrix operations        | High power, high inference latency            |

GPUs are optimized for batch processing—running many operations in parallel across large datasets. What we need is single-event, microsecond-level inference on a continuous stream of order book snapshots. That's not what GPUs are designed for.

FPGAs let you build a custom pipeline that matches the exact computation you need. Less flexibility, more speed.

---

## What I Set Out to Build

The goal: embed a trained LSTM directly into FPGA hardware so it can classify order book behavior in real time—acting as a "reflex in silicon" that flags microstructure events before they materially impact price.

The specific targets:
- Inference latency under 3 microseconds
- Throughput above 1 million frames per second
- Run on a Xilinx Zynq-7020 FPGA using the FINN compiler framework

**Part 2** covers the simulator I built to generate training data, the model architecture, the FPGA compilation pipeline, the actual results, and—honestly—where the implementation hit walls.

---

*This is Part 1 of 2. [Read Part 2 here.](/posts/fpga-lstm-hft-part-2)*
