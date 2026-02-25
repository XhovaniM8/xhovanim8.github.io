---
title: "I Tried to Put an AI Inside Trading Hardware (Part 2: The Implementation)"
date: 2026-02-24 14:00:00 +0000
categories: [projects, research]
tags:
  [
    fpga,
    lstm,
    high-frequency-trading,
    order-book,
    finn,
    brevitas,
    quantization,
    deep-learning,
    nyu,
    capstone,
    quant,
    hls,
  ]
description: "Part 2 of my NYU capstone: building a quantized LSTM for FPGA deployment using Brevitas and FINN. Covers the order book simulator, model training, FINN compilation pipeline, hardware estimates, and where it broke down."
---

*This is Part 2. [Part 1 covers the problem and finance context.](/posts/fpga-lstm-hft-part-1)*

---

## The Plan

Train a compact LSTM to classify order book behavior. Quantize it to 8 bits. Compile it to FPGA hardware using FINN. Get sub-microsecond inference latency.

That's the plan. Here's what actually happened.

---

## Step 1: Generating Training Data

I don't have access to live Level 2 order book data at the quality needed for this kind of work. That data is expensive and proprietary.

So I built a simulator in C++.

Each tick, the simulator generates a price update with a slight upward bias—markets don't purely random-walk—and then with 20% probability fires a random microstructure event:

```cpp
void OrderbookSimulator::generateUpdate() {
    double timeDelta = /* elapsed ms */ / 1000.0;

    // 60% chance of upward drift, otherwise down
    double drift = uniform_dist(rng) < 0.6 ? +0.003 : -0.003;
    double noise = normalDist(rng) * std::sqrt(timeDelta);
    currentPrice = std::max(currentPrice + drift + noise, tickSize);

    // Refresh all price levels
    for (int i = 1; i <= numLevels; ++i) {
        double bidPrice = currentPrice - i * tickSize;
        double askPrice = currentPrice + i * tickSize;
        orderbook.updateBid(bidPrice, /* size decays with depth */);
        orderbook.updateAsk(askPrice, /* size decays with depth */);
    }
}
```

The interesting part is the event set. Eight distinct behaviors, each with a realistic mechanism:

**Spoofing** — places a 10× oversized order then silently removes it 50ms later in a detached thread:

```cpp
case SPOOF:
    double spoofSize = askLevels[level].volume * 10.0;
    orderbook.updateAsk(spoofPrice, spoofSize);  // Place

    std::thread([=, this]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        orderbook.updateAsk(spoofPrice, askLevels[level].volume);  // Revert
    }).detach();
    break;
```

**Liquidity sweep** — aggressively clears 3–7 levels from one side and moves price accordingly:

```cpp
case SWEEP:
    int sweepDepth = std::uniform_int_distribution<int>(3, 7)(rng);
    for (int i = 0; i < sweepDepth && i < askLevels.size(); ++i) {
        orderbook.updateAsk(askLevels[i].price, 0.0);  // Zero out levels
    }
    currentPrice += tickSize * sweepDepth * 2.0;
    break;
```

**Cancellations, large orders, shifts** — simpler variants that reduce volume to 10%, multiply by 2–5×, or jump the price by 3 ticks.

The key advantage: every event gets a label. When real market data shows a spoofing pattern it isn't labeled—it takes surveillance teams months to identify them. In simulation, you know exactly what you injected.

---

## Step 2: Feature Engineering

The `FeatureExtractor` class converts each order book snapshot into a flat vector. The features fall into three groups.

**Instantaneous** — computed directly from the current state:

```cpp
feature.spread    = state.spread;
feature.spreadPct = state.spread / state.midPrice;
feature.priceChange = (state.midPrice - prevPrice) / prevPrice;

// Order flow imbalance: positive = more bid pressure
double totalBidSize = state.bestBid.second;
double totalAskSize = state.bestAsk.second;
feature.sizeImbalance = (totalBidSize - totalAskSize) / (totalBidSize + totalAskSize);

// Volume-weighted mid-price vs conventional mid-price
feature.vwmp = (state.bestBid.first * state.bestAsk.second +
                state.bestAsk.first * state.bestBid.second) /
               (state.bestBid.second + state.bestAsk.second);
feature.vwmpDiff = (feature.vwmp - state.midPrice) / state.midPrice;
```

**Structural** — for each of the top 5 price levels on each side:

```cpp
feature.bidDistances[i] = (state.midPrice - state.bidLevels[i].price) / state.midPrice;
feature.bidSizesNorm[i] = state.bidLevels[i].volume / volumeNormalization;
// same for ask side
```

**Rolling** — computed over a window of 10 prior snapshots: volatility (std dev of price changes), momentum at 1/5/10 steps, smoothed price and spread trends.

Total: **32 features per snapshot**. Packed into a sequence of 10 timesteps = **320 elements per sample**.

Labeling uses a 5-step horizon and a 5 basis point threshold:

```cpp
double futureReturn = (futurePrice - currentPrice) / currentPrice;

if (futureReturn > 0.0005)       label = 0;  // Up
else if (futureReturn < -0.0005) label = 1;  // Down
else                              label = 2;  // No Change
```

The class distribution is brutal. From the actual run:

```
Class distribution: Up=884, Down=412, No Change=59423
```

That's ~98.8% "No Change." A model that predicts No Change for everything gets 98.8% accuracy and is completely useless for trading.

To address it: oversample minority classes to 3,000 samples each, and use inverse-frequency class weights in the loss function:

```python
class_counts = torch.tensor([884, 412, 59423], dtype=torch.float32)
inv_freqs = 1.0 / class_counts
weights = inv_freqs / inv_freqs.sum()
criterion = nn.CrossEntropyLoss(weight=weights)
```

---

## Step 3: The Model

The LSTM architecture is intentionally small. FPGA resources are limited, and complexity is the enemy of low latency.

Brevitas is a PyTorch extension that adds quantization-aware layers. You define the model almost identically to a normal PyTorch model, but swap `nn.LSTM` and `nn.Linear` for `QuantLSTM` and `QuantLinear`, passing in quantization specs:

```python
from brevitas.nn import QuantLSTM, QuantLinear
from brevitas.quant import Int8ActPerTensorFixedPoint, Int8WeightPerTensorFixedPoint

class QuantLSTMModel(nn.Module):
    def __init__(self, input_size, hidden_size, num_classes):
        super().__init__()
        self.lstm = QuantLSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            batch_first=True,
            weight_quant=Int8WeightPerTensorFixedPoint,
            io_quant=Int8ActPerTensorFixedPoint
        )
        self.fc = QuantLinear(
            in_features=hidden_size,
            out_features=num_classes,
            bias=True,
            weight_quant=Int8WeightPerTensorFixedPoint,
            act_quant=Int8ActPerTensorFixedPoint
        )

    def forward(self, x):
        out, _ = self.lstm(x)
        return self.fc(out[:, -1, :])  # Last timestep only
```

Input shape is `[batch, 10, 32]`—10 timesteps, 32 features each. The LSTM has 64 hidden units. Output is 3-class logits.

During the forward pass Brevitas simulates fixed-point 8-bit arithmetic, including quantization noise. Gradients still flow through in full precision. The result is a model that trains like a float32 network but already knows it's going to run as int8 on hardware.

Training used Adam at lr=1e-3, macro F1 as the early-stopping metric (not accuracy, for the class imbalance reason), and saved the best checkpoint:

```python
if f1 > best_f1:
    best_f1 = f1
    torch.save(model.state_dict(), "best_model.pth")
```

---

## Step 4: The FPGA Compilation Pipeline

This is where things get interesting—and where things also eventually broke.

The workflow starts with exporting the trained model to ONNX:

```python
model.eval()
dummy_input = torch.randn(1, sequence_length, feature_dim)

torch.onnx.export(
    model,
    dummy_input,
    "quant_lstm.onnx",
    opset_version=11,
    input_names=["input"],
    output_names=["output"]
)
```

This gives you a standard ONNX graph. FINN then takes that graph through its own transformation pipeline to produce synthesizable hardware.

The workflow:

1. **Export to QONNX** — Quantized ONNX format that preserves quantization metadata for hardware synthesis
2. **FINN transformations** — A 10-step pipeline that converts the neural network graph into synthesizable hardware

The FINN transformation pipeline, roughly in order:

- **Tidy-up pass:** Shape inference, constant folding, tensor renaming, removing static inputs
- **Input annotation:** Mark the input tensor as UINT8
- **Streamline:** Reorganize linear operations (fuse multiplications/additions) to simplify graph topology
- **Scalar folding:** Absorb scalar ops into the TopK layer
- **Hardware conversion:** Transform fully connected and activation layers into Matrix-Vector-Activation Units (MVAUs) and thresholding units
- **StreamingDataflowPartition:** Isolate hardware-compatible nodes for standalone synthesis
- **SpecializeLayers:** Replace generic layer types with `hls` counterparts for HLS code generation
- **SetFolding:** Configure Processing Elements (PEs) and SIMD lanes to hit the 1M FPS throughput target

With full synthesis, this generates RTL that you flash to the FPGA. I ran the estimate-only flow, which simulates hardware performance without triggering full synthesis.

---

## Results

### Classification Performance

The model achieves a **macro F1 score of 0.41** on the test set.

That sounds low. For context: a naive model that always predicts "No Change" gets ~98% accuracy but 0% recall on the signals that actually matter. Macro F1 averages performance equally across all three classes regardless of their frequency—it's the right metric here.

Confusion matrix highlights:
- "No Change" class: 93% precision, handles the volume well
- "Up" and "Down": 52% and 47% recall—the model catches roughly half the directional moves
- Misclassifications mostly go to adjacent classes (Up → No Change, not Up → Down)

The t-SNE visualization of LSTM hidden states shows distinct clustering by movement direction—Up and Down form separate clusters, No Change spans a wider region. The model is learning meaningful latent representations. The classification errors reflect genuine ambiguity in the data, not random noise.

### Hardware Estimates

| Metric     | Value                                  |
| ---------- | -------------------------------------- |
| Latency    | 2.52 microseconds                      |
| Throughput | 1,020,408 frames per second            |
| LUT usage  | 38,128 / 53,200 on Zynq-7020 (71.7%)  |
| BRAM usage | 66 / 140 (47.1%)                       |
| DSP usage  | 0 (purely LUT-based arithmetic)        |

The critical layer is `MVAU_hls_0` at 98 clock cycles. The projected latency of 2.52μs is consistent with published FPGA-LSTM results in the literature and falls within the microsecond-level requirements of modern HFT systems.

**Platform comparison:**

| Platform        | Latency (μs) | Throughput (FPS) | Power (W) |
| --------------- | ------------ | ---------------- | --------- |
| FPGA (estimated)| 2.52         | 1,020,408        | ~5        |
| CPU (Xeon E5)   | 85.3         | 11,723           | ~95       |
| GPU (T4)        | 12.7         | 78,740           | ~70       |
| ASIC (est.)     | 0.85         | 1,176,470        | ~2        |

For reference: a 10GbE link transmits one Ethernet frame in roughly 1 microsecond. A 2.52μs inference latency means the model adds about two frame transmissions of delay—feasible for integration into a wire-speed trading pipeline.

---

## Where It Broke Down

Being honest about the failures:

**Constant folding took 30+ minutes.** LSTM unrolling produces 1,400+ nodes in the ONNX graph. Constant folding over that structure is slow. Reducing sequence length and hidden size would help.

**Full hardware synthesis didn't complete.** Vivado and Vitis were installed and functional. The LSTM's sequential dependencies created optimization challenges during HLS compilation that I didn't resolve within the project scope.

**Shape mismatch errors.** The persistent blocker. After model transformations, FINN expected the input as `[1, 320]`—a flattened vector. The exported model retained its original `[1, 5, 64]` rank. Manually inserting a Reshape node caused ONNX runtime errors at the Mul operation during constant folding.

The fix would be to enforce the flattened shape earlier—either during Brevitas export or before quantization—rather than trying to reshape downstream in the FINN pipeline. That's the correct engineering answer, and it's documented for the next iteration.

**Limited debug visibility.** FINN transformations give minimal progress feedback during long-running phases. Diagnosing where exactly the shape propagation failed required digging into the ONNX graph manually.

---

## What I'd Do Differently

1. **Lock the input shape before export.** Force the model to accept `[1, 320]` from the start, not `[1, 5, 64]`. Saves the reshape headache entirely.

2. **Smaller model first.** Validate the full synthesis pipeline with a trivially simple LSTM (2–4 hidden units) before scaling up. The estimate flow working doesn't mean synthesis will.

3. **Reduce ONNX node count.** Explore splitting LSTM cells into primitive operations that FINN handles more cleanly. The framework has better native support for feedforward layers; LSTM's feedback paths are harder.

4. **Consider hybrid architectures.** A 1D CNN frontend for spatial feature extraction followed by a smaller LSTM for temporal modeling might achieve better accuracy with lower FPGA resource requirements.

---

## Future Directions

Things I'd want to build from here:

- **Multi-asset processing:** Analyze correlated instruments simultaneously. Preliminary analysis suggests 3–4× higher throughput compared to parallel single-asset deployments.
- **Online adaptation:** Periodic CPU-side weight updates pushed to FPGA parameters. Keeps the model current without full retraining.
- **Real exchange integration:** Live parsers for NASDAQ ITCH or NYSE XDP protocols, connected directly to signal generation and order management.
- **Integrated risk controls:** Pre-trade checks (position limits, kill switches) co-designed into the FPGA pipeline so they don't add latency.

---

## What This Actually Demonstrates

The full synthesis didn't finish. That's worth being clear about.

What did work:
- A quantized LSTM model that learns meaningful representations of order book microstructure
- Successful completion of the FINN estimate-only pipeline with realistic hardware projections
- Projected latency and throughput consistent with published FPGA-LSTM literature
- A clear diagnosis of where the implementation stalled and what the correct fix is

The broader point stands: quantized LSTM models can theoretically be compiled to FPGA hardware with sub-3-microsecond inference latency using existing open-source toolchains. The barriers are engineering, not fundamental. FINN's own academic tutorials demonstrate working sub-microsecond inference on Zynq-7000 series FPGAs for simple quantized networks.

The gap between "simple quantized feedforward network" and "LSTM with temporal state" is real, and it's where this project ran out of runway.

---

*Full paper available upon request. Source code for the LSTM model and synthetic data generator is available for academic purposes.*
