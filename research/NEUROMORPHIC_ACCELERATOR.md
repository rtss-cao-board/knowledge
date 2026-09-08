# USB Neuromorphic Accelerator: Feasibility Study & Architecture Design

**Author:** Turing (RTSS Board)
**Date:** September 2026
**Status:** Feasibility Study
**Classification:** RTSS Research

---

## Executive Summary

This document evaluates the feasibility of building a peripheral neuromorphic accelerator that connects to standard computers via USB-C or M.2. We analyze existing commercial options, assess the FPGA-based custom design path, define a target architecture, and estimate cost/performance/power for a prototype.

**Conclusion:** A custom FPGA-based neuromorphic peripheral is technically feasible but economically unjustified. BrainChip's Akida AKD1500 M.2 module ($249) already provides a production-quality neuromorphic accelerator in the exact form factor we'd be designing. The build-vs-buy analysis strongly favors buy for any near-term deployment. A custom design only makes sense if targeting a neuron model or spike routing topology that Akida doesn't support.

---

## Part 1: Existing Commercial Options

### BrainChip Akida (Buy Option)

The AKD1000/AKD1500 is the closest thing to what we'd build:

| Spec | AKD1000 | AKD1500 (July 2026) |
|------|---------|---------------------|
| Form factor | M.2 B+M key | M.2 B+M key |
| Interface | PCIe Gen 2 x1 | PCIe Gen 2 x1 |
| Power | 300 mW typical | ~200 mW typical |
| Throughput | 800 GOPS | Improved (TBD) |
| Neuron model | Event-based CNN layers | Event-based CNN + temporal |
| Price | $249 (M.2 module) | ~$249 (estimated) |
| SDK | MetaTF (Python, Keras-like) | MetaTF 2.0 |
| Availability | DigiKey, direct | DigiKey, direct |

**Pros:** Production quality, real SDK, M.2 plug-and-play, sub-watt power, $249.
**Cons:** Proprietary neuron model (not LIF-based SNN), limited to Akida's CNN-like architecture. Not true spiking in the biological sense.

### Intel Loihi 2 (Research Option)

| Spec | Value |
|------|-------|
| Neurons | 1M programmable |
| Synapses | 120M |
| Process | Intel 4 (EUV) |
| Neuron model | Fully programmable via microcode |
| Interface | Research systems only (Kapoho Point USB, Oheo Gulch) |
| Price | Not commercially available |
| SDK | Lava framework (open source) |
| Availability | Intel Neuromorphic Research Community (application required) |

**Pros:** Most flexible neuron model, true spiking, programmable learning rules, Intel backing.
**Cons:** Not commercially available. Research access only. No M.2/USB product.

### SpiNNaker 2 (Research Option)

| Spec | Value |
|------|-------|
| Cores | 152 ARM Cortex-M4F per chip |
| Process | 22nm FDSOI |
| Neuron model | Any model expressible in C |
| Interface | Custom boards, research systems |
| Price | Not commercially available |
| Availability | University of Manchester research program |

**Pros:** Maximum flexibility (ARM cores), any neuron model.
**Cons:** Not commercially available. Research only. Not energy-optimal.

---

## Part 2: Custom FPGA Design Path

### Why Build Custom?

The only justification for a custom peripheral over buying Akida:
1. Need true LIF/Izhikevich spiking neurons (Akida uses event-based CNN, not biological SNN)
2. Need custom spike routing topology (mesh, crossbar, small-world)
3. Need on-chip learning (STDP) not supported by Akida
4. Research/education platform with open architecture
5. Cost reduction at volume (thousands of units)

### Target FPGA Platforms

| FPGA | LUTs | BRAM | Price | Power | USB Support |
|------|------|------|-------|-------|-------------|
| Lattice iCE40 UP5K | 5,280 | 120 Kb | $6 | 1-10 mW | Via FTDI bridge |
| Lattice ECP5-25F | 24,000 | 1,008 Kb | $15 | 50-200 mW | Via FTDI bridge |
| Xilinx Spartan-7 XC7S25 | 14,600 | 1,620 Kb | $20 | 100-500 mW | Via FT2232H |
| Xilinx Artix-7 XC7A35T | 33,280 | 1,800 Kb | $35 | 200-800 mW | Native via soft core |
| Gowin GW2A-18 | 20,736 | 828 Kb | $12 | 50-300 mW | Via FTDI bridge |

### LIF Neuron Resource Cost

A single LIF neuron in digital logic requires:
- Membrane potential register: 16-bit fixed-point = 16 flip-flops
- Threshold comparator: ~16 LUTs
- Decay multiplier: ~32 LUTs (fixed-point multiply by beta)
- Reset logic: ~4 LUTs
- Spike output register: 1 flip-flop
- **Total per neuron: ~52 LUTs + 17 FFs**

With synaptic weight storage in BRAM (8-bit weights):
- 256 input synapses × 8 bits = 256 bytes per neuron
- BRAM access: shared across time-multiplexed neurons

### Neuron Capacity by FPGA

| FPGA | LUTs | Max Parallel Neurons | Time-Multiplexed (8:1) | Synapses (BRAM) |
|------|------|---------------------|----------------------|-----------------|
| iCE40 UP5K | 5,280 | 101 | 808 | 15K (120Kb) |
| ECP5-25F | 24,000 | 461 | 3,692 | 129K (1,008Kb) |
| Spartan-7 | 14,600 | 280 | 2,246 | 207K (1,620Kb) |
| Artix-7 35T | 33,280 | 640 | 5,120 | 230K (1,800Kb) |

Time-multiplexing: process 8 neurons per clock cycle on the same hardware by rotating through neuron states stored in BRAM.

### Comparison to Existing Chips

| Platform | Neurons | Synapses | Power | Price | Flexibility |
|----------|---------|----------|-------|-------|-------------|
| Our FPGA (Artix-7) | 5,120 | 230K | ~500mW | ~$80 BOM | Full (open HDL) |
| BrainChip Akida | Commercial | Commercial | 300mW | $249 | SDK only |
| Intel Loihi 2 | 1M | 120M | Variable | N/A | Research only |
| SpiNNaker 2 | 152K | 152M | Adaptive | N/A | Research only |

Our FPGA design would have **200x fewer neurons** than Loihi 2 and **30x fewer** than SpiNNaker 2 per chip. But it would be open, commercially available, and cost $80 in components.

---

## Part 3: Architecture Design

### Block Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    HOST COMPUTER                         │
│                                                         │
│   Python SDK ──► USB-C ──► FPGA Board                   │
│   (load weights,          (neuromorphic core)           │
│    stream spikes,                                       │
│    read outputs)                                        │
└─────────────┬───────────────────────────────────────────┘
              │ USB 2.0 / USB 3.0
              │ (Bulk transfer mode)
┌─────────────▼───────────────────────────────────────────┐
│                   FPGA BOARD                             │
│                                                         │
│  ┌──────────┐  ┌──────────────────────────────────────┐ │
│  │   USB    │  │         NEUROMORPHIC CORE             │ │
│  │  Bridge  │──│                                      │ │
│  │ (FTDI /  │  │  ┌────────┐  ┌────────┐  ┌───────┐ │ │
│  │  soft)   │  │  │Neuron  │  │Spike   │  │Weight │ │ │
│  │          │  │  │Array   │──│Router  │──│Memory │ │ │
│  └──────────┘  │  │(LIF ×N)│  │(Xbar)  │  │(BRAM) │ │ │
│                │  └────────┘  └────────┘  └───────┘ │ │
│  ┌──────────┐  │                                      │ │
│  │  Config  │  │  ┌────────┐  ┌────────┐             │ │
│  │  Memory  │──│  │Input   │  │Output  │             │ │
│  │  (Flash) │  │  │Encoder │  │Decoder │             │ │
│  └──────────┘  │  │(Rate)  │  │(Spike  │             │ │
│                │  └────────┘  │ Count) │             │ │
│                │              └────────┘             │ │
│                └──────────────────────────────────────┘ │
│                                                         │
│  Power: USB bus-powered (500mA @ 5V = 2.5W budget)     │
└─────────────────────────────────────────────────────────┘
```

### Component Breakdown

**1. USB Bridge**
- FTDI FT2232H dual-channel USB 2.0 (480 Mbps)
- Channel A: Spike data stream (bulk transfer)
- Channel B: Configuration/control (command interface)
- Cost: $4.50

**2. Neuromorphic Core**
- N LIF neurons with configurable beta (decay), threshold, reset mode
- Time-multiplexed: 8 virtual neurons per physical neuron circuit
- Each neuron: 16-bit membrane potential, 8-bit weights
- Spike generated when membrane > threshold
- Membrane reset to zero (hard reset) or subtract threshold (soft reset)

**3. Spike Router**
- Crossbar switch connecting any neuron to any other
- Configurable connectivity matrix loaded from host
- Supports broadcast (one-to-many) and convergence (many-to-one)
- Address-Event Representation (AER) protocol: each spike is a (neuron_id, timestamp) tuple

**4. Weight Memory**
- BRAM blocks store synaptic weights (8-bit signed)
- Organized as weight_matrix[source_neuron][target_neuron]
- Loaded from host via USB at initialization
- Optional: STDP update logic for on-chip learning

**5. Input Encoder**
- Rate coding: host sends analog values, encoder generates Poisson spike trains
- Direct spike injection: host sends pre-computed spike times
- Configurable encoding mode per input channel

**6. Output Decoder**
- Spike counter per output neuron over configurable time window
- Raw spike stream output for host-side processing
- Configurable readout: spike count, first-spike time, or membrane potential

### Programming Model

```python
import neuroaccel as na

# Connect to USB device
device = na.connect()  # Auto-detect USB neuromorphic accelerator

# Define network
net = na.Network()
layer1 = net.add_layer(neurons=256, beta=0.9, threshold=1.0)
layer2 = net.add_layer(neurons=10, beta=0.9, threshold=1.0)
net.connect(layer1, layer2, weights=weight_matrix)  # 256x10 int8

# Upload to device
device.load(net)

# Run inference
input_data = np.array([...])  # 256 float values
spikes_out = device.run(input_data, timesteps=25)  # Returns spike counts per output neuron
prediction = spikes_out.argmax()

# Streaming mode (for video/sensor data)
for frame in camera_stream:
    features = extract_features(frame)  # CPU preprocessing
    result = device.run(features, timesteps=10)
    process(result)
```

### Interface Protocol

**USB Bulk Transfer Packets:**

```
Command packet (host → device):
  [0x01] LOAD_WEIGHTS   : [layer_id, weight_data...]
  [0x02] SET_PARAMS      : [layer_id, beta, threshold, reset_mode]
  [0x03] INJECT_SPIKES   : [timestep, neuron_id, neuron_id, ...]
  [0x04] RUN             : [num_timesteps, input_data...]
  [0x05] READ_OUTPUT     : [mode (count/stream/membrane)]
  [0x06] SET_CONNECTIVITY : [source_layer, target_layer, mask...]

Response packet (device → host):
  [0x81] SPIKE_COUNTS    : [neuron_0_count, neuron_1_count, ...]
  [0x82] SPIKE_STREAM    : [timestamp, neuron_id, timestamp, neuron_id, ...]
  [0x83] MEMBRANE_STATE  : [neuron_0_potential, neuron_1_potential, ...]
  [0x84] STATUS          : [state, neurons_active, spikes_this_run, power_mw]
```

---

## Part 4: BOM Estimate (Prototype)

| Component | Part | Qty | Unit Cost | Total |
|-----------|------|-----|-----------|-------|
| FPGA | Xilinx Artix-7 XC7A35T | 1 | $35 | $35 |
| USB Bridge | FTDI FT2232H | 1 | $4.50 | $4.50 |
| Flash | Winbond W25Q128 (16MB) | 1 | $2 | $2 |
| Voltage regulators | 3.3V, 1.0V, 1.8V LDOs | 3 | $1 | $3 |
| Oscillator | 100 MHz MEMS | 1 | $1.50 | $1.50 |
| PCB | 4-layer, 50x30mm | 1 | $15 | $15 |
| Passives | Caps, resistors | ~30 | $0.05 | $1.50 |
| USB-C connector | USB-C receptacle | 1 | $0.50 | $0.50 |
| Assembly | PCBA (prototype) | 1 | $20 | $20 |
| **Total** | | | | **$83** |

**Alternative: M.2 form factor**
Replace USB-C connector with M.2 edge connector (+$2), add PCIe soft core to FPGA (uses LUTs). Total: ~$85.

### Production Cost at Volume

| Volume | Unit Cost | Notes |
|--------|-----------|-------|
| 1 (prototype) | $83 | Hand-assembled |
| 10 | $65 | Small batch PCBA |
| 100 | $45 | Batch production |
| 1000 | $30 | Production pricing on FPGA |

---

## Part 5: Performance Estimate

### Clock and Throughput

FPGA clock: 100 MHz
Neurons per clock (time-multiplexed): 8
Neuron updates per second: 800M

For a 5,120-neuron network with 25 timesteps:
- Total neuron updates: 5,120 × 25 = 128,000
- Time: 128,000 / 800M = 0.16 ms
- **Inference latency: ~0.16 ms**

For comparison:
- Our CPU SNN (ARM64): 0.75 ms for 102K params, 25 timesteps
- BrainChip Akida: sub-millisecond for supported models
- Intel Loihi 2: microsecond-scale for on-chip networks

### USB Bandwidth

USB 2.0 bulk: ~40 MB/s effective
Input data per inference: 5,120 neurons × 2 bytes = 10 KB
Output data per inference: 10 neurons × 2 bytes = 20 bytes
Transfer overhead: ~0.3 ms

**Total inference (including USB): ~0.5 ms**

USB 3.0 would reduce transfer to ~0.03 ms but requires more complex bridge ($8 vs $4.50).

### Power

FPGA core: ~200 mW at 100 MHz
USB bridge: ~100 mW
Regulators/misc: ~50 mW
**Total: ~350 mW** (USB bus-powered, well within 2.5W USB budget)

---

## Part 6: Build vs Buy Analysis

| Criteria | Custom FPGA | BrainChip Akida M.2 |
|----------|-------------|---------------------|
| Price (prototype) | $83 | $249 |
| Price (100 units) | $45 | $249 |
| Neurons | 5,120 | Proprietary (larger) |
| True LIF spiking | Yes | No (event-based CNN) |
| STDP on-chip learning | Possible | No |
| Custom neuron models | Yes (HDL) | No |
| SDK maturity | None (build from scratch) | MetaTF (mature) |
| Time to first inference | 3-6 months | 1 day |
| PCIe/M.2 support | Soft core needed | Native |
| Production readiness | No | Yes |
| Open source | Yes | No |

### Verdict

**For Sentinel deployment: Buy Akida.** The $249 M.2 module plugs into the Jetson's M.2 slot and provides immediate neuromorphic acceleration with a mature SDK. Time-to-deployment: days, not months.

**For RTSS research: Build custom.** An open FPGA neuromorphic accelerator enables experiments that Akida can't support: custom neuron models, STDP learning, novel routing topologies. The $83 prototype cost is low enough to justify as a research platform. The open HDL becomes publishable and potentially a product if the research yields results worth deploying.

**Recommended path:**
1. Buy Akida M.2 now ($249) for Sentinel integration experiments
2. Design and prototype the FPGA accelerator in parallel (~3 months)
3. Compare real-world performance on Sentinel workloads
4. Decide whether the custom hardware offers enough advantage to justify production

---

## Part 7: Next Steps

### If Building Custom:
1. Select FPGA dev board (Digilent Arty A7-35T, $130, has Artix-7 + USB + PMOD)
2. Implement LIF neuron array in Verilog/VHDL
3. Implement crossbar spike router
4. Implement USB command protocol
5. Write Python SDK
6. Benchmark against CPU SNN and Akida
7. Publish results as research paper

### If Buying Akida:
1. Order AKD1000 M.2 module ($249 from BrainChip shop or DigiKey)
2. Install in Jetson Orin Nano Super M.2 slot
3. Install MetaTF SDK
4. Convert Sentinel's detection model to Akida-compatible format
5. Benchmark inference latency and power vs TensorRT
6. Publish comparison paper

### Timeline
- Month 1: Akida procurement and initial benchmarks
- Month 1-3: FPGA design and HDL implementation
- Month 3: Side-by-side comparison paper
- Month 4+: Decision on production path
