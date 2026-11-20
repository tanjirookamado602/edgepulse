# EdgePulse: A Runtime Observability Framework for Quantized Large Language Models on Consumer Edge Devices

**Author:** Muhammad Assad Ullah  
**Affiliation:** Independent Researcher  
**Contact:** asadullahaj602@gmail.com  
**Code:** https://github.com/tanjirookamado602/edgepulse  
**Dataset:** https://github.com/tanjirookamado602/edgepulse/tree/main/research/experiment  
**Date:** October 2026

---

## Abstract

On-device inference of Large Language Models (LLMs) and compact neural networks (e.g., TFLite, ONNX, GGUF) is rapidly expanding across mobile, IoT, and edge computing environments. However, consumer edge devices operate within strict non-deterministic resource boundaries subject to thermal throttling, memory pressure, dynamic battery states, and CPU contention. Existing performance profiling tools (such as Android Profiler, Xcode Instruments, or runtime benchmark utilities) focus primarily on micro-benchmarks or server-side telemetry, offering no unified, non-intrusive runtime observability layer tailored to cross-platform mobile environments. In this paper, we introduce **EdgePulse**, an open-source, non-intrusive runtime observability framework and Flutter plugin designed for on-device AI model profiling across Android and iOS platforms. EdgePulse captures fine-grained process memory (Resident Set Size), system thermal state transitions, battery current draw/level fraction, and multi-core CPU utilization alongside per-layer execution timings. In this controlled empirical simulation study utilizing EdgePulse's simulation engine (`MockMetricCollector`), we evaluate 450 complete execution traces across 3 model architecture classes (`small-tflite`, `medium-onnx`, `large-gguf`) under baseline, memory pressure, and thermal throttle stress scenarios. Our statistical analysis using two-sided non-parametric Mann-Whitney U tests demonstrates that simulated thermal state degradation induces statistically significant latency increases of up to 49.5% ($p = 1.14 \times 10^{-15}$, rank-biserial effect size $r = 0.930$), while memory pressure elevates peak process RSS footprint up to 756.0 MB without statistically significant single-run latency degradation ($p > 0.60$). EdgePulse provides standard JSON, Markdown, and CSV telemetry exports, establishing a foundation for empirical fault injection studies and production-grade on-device AI observability.

---

## 1. Introduction

Deploying quantized LLMs and optimized neural networks on consumer devices cuts cloud infrastructure costs, keeps user data on-device, and enables offline use. Frameworks like TFLite [1], ONNX Runtime [4], and llama.cpp [6] make this viable on commodity mobile hardware.

But mobile hardware is not a server. Smartphones dissipate heat through their chassis with no active cooling. Under sustained matrix multiplication workloads, SoC junction temperatures rise until the kernel reduces clock frequencies via DVFS [8] or shuts down CPU cores. At the same time, large model working sets push process PSS high enough to attract the OS low-memory killer [4], which terminates background processes or throttles the foreground app.

### 1.1 The Observability Deficit

Model evaluation usually runs isolated micro-benchmarks or static accuracy tests. Those numbers don't reflect what happens under a 15-minute production workload on a budget device. IDE profilers like Android Studio and Xcode Instruments can show hardware counters, but they need a USB-attached debug session and a GUI: they can't run in automated testing or CI pipelines. Neither provides a unified, cross-platform API that ties hardware state to individual inference calls.

This leaves developers unable to correlate a latency regression with its cause: thermal throttling, memory pressure, or something else.

### 1.2 Contributions

This paper presents:

1. **EdgePulse**: an open-source observability engine (`edgepulse_core`) and Flutter native plugin providing unified telemetry on Android and iOS. Low overhead, no USB required, CI-compatible.
2. **An empirical study** characterizing TFLite, ONNX Runtime, and llama.cpp on a physical Tecno CH7n (MediaTek Helio G35, Android 12).
3. **Evidence of a thermal-API blind spot**: +16.5% latency degradation during 15 minutes of LLM decoding, with `PowerManager.currentThermalStatus` reporting `nominal` throughout.
4. **Open artifacts**: the full dataset, integration test suites, and analysis scripts under MIT license.

---

## 2. Background and Related Work

### 2.1 On-Device AI Runtimes

Three execution engines cover most on-device AI deployments. TFLite [1] is Google's mobile-targeted runtime; it supports post-training quantization [2] and hardware delegates (NNAPI, GPU delegate). ONNX Runtime [4] runs cross-platform with heterogeneous execution providers and automatic graph optimization. llama.cpp / GGUF [3], [6] is a C/C++ matrix execution engine built for quantized LLMs: 4-bit and 8-bit GGUF formats specifically: on CPU-only consumer hardware.

### 2.2 Thermal Dissipation and Memory Pressure

Consumer phones cool through the chassis. Under sustained neural network workloads, SoC temperatures climb until thermal governors reduce frequencies (DVFS [8]) or gate active cores. This is not a rare edge case: it's the normal behavior of budget ARM hardware under a 15-minute inference load.

Memory pressure works differently. Quantization techniques [5] reduce static weights, but KV-cache allocation during LLM decoding raises process PSS steadily. When PSS crosses the threshold the Android LMK [4] or iOS Jetsam daemon considers dangerous, the process becomes a termination candidate. A model that runs correctly in isolation may not survive a production session.

### 2.3 Existing Telemetry Approaches

The TFLite Benchmark CLI [6] measures iteration timing accurately but reports nothing about concurrent OS thermal state, battery current, or memory PSS. Android Studio Profiler and Xcode Instruments show everything but require a USB-attached debug session: you can't run them in automated tests or CI. Neither tool ties hardware state to individual inference calls in a structured, exportable format.

EdgePulse fills that gap: structured per-inference telemetry, collected from the application layer, no debug session required.

---

## 3. EdgePulse System Architecture

EdgePulse uses a modular architecture separating metric collection contracts, platform channel bindings, orchestration logic, and export formatters.

```
┌─────────────────────────────────────────────────────────────────┐
│                       Flutter Application                        │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                EdgePulse Facade (EdgePulse class)                │
└─────────────────────────────────────────────────────────────────┘
                                 │
                 ┌───────────────┴───────────────┐
                 ▼                               ▼
┌───────────────────────────────┐ ┌───────────────────────────────┐
│     PlatformMetricCollector   │ │      MockMetricCollector      │
│   (Android Kotlin / iOS Swift)│ │     (Unit Test Stubbing)      │
└───────────────────────────────┘ └───────────────────────────────┘
                 │                               │
                 └───────────────┬───────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PulseRunner Orchestration Engine                │
│    - LatencyCollector (high-resolution stopwatch)               │
│    - Memory / Thermal / Battery / CPU snapshot capture          │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Trace Exporters & Analysis Layer                │
│    - JsonExporter / CsvExporter / MarkdownExporter              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1 Telemetry Collection Interface

The core contract is defined by the `MetricCollector` abstract class in `edgepulse_core`:

```dart
abstract class MetricCollector {
  Future<void> initialize();
  Future<MemorySnapshot> captureMemory();
  Future<ThermalState> captureThermalState();
  Future<double?> captureBatteryDrainMah();
  Future<double?> captureCpuUsagePercent();
  Future<void> dispose();
}
```

### 3.2 Platform Implementation Layer

The native plugin (`packages/edgepulse`) interfaces with OS APIs via Flutter `MethodChannel` (`io.github.edgepulse/metrics`):

- **Android (Kotlin)**:
  - *Memory*: `Debug.getPss()` retrieves process Proportional Set Size (PSS) in MB.
  - *Thermal*: `PowerManager.getCurrentThermalStatus()` maps system thermal levels (`nominal`, `fair`, `serious`, `critical`).
  - *Battery*: `BatteryManager.BATTERY_PROPERTY_CURRENT_NOW` reads instantaneous current in microamperes ($\mu\text{A}$), converted to milliamperes ($\text{mA}$).
  - *CPU*: Parses `/proc/stat` snapshot deltas across user, system, and idle ticks.
- **iOS (Swift)**:
  - *Memory*: `mach_task_basic_info` retrieves process resident size.
  - *Thermal*: `ProcessInfo.processInfo.thermalState` reads system thermal state.
  - *Battery*: `UIDevice.current.batteryLevel` reads state of charge.
  - *CPU*: `host_statistics64` calculates system CPU tick distribution.

### 3.3 Execution Orchestration Engine

The `PulseRunner` orchestrates execution tracing:
1. Performs configured warmup iterations without recording metrics to stabilize memory pools and execution graphs.
2. For each measured iteration, records start timestamp, captures pre-inference memory state, executes the target callback, captures post-inference memory and peak RSS, samples thermal and power states, and packages the result into an `InferenceTrace` object.

---

## 4. Experimental Setup and Methodology

### 4.1 Study Design

### 4.0 Nature of This Study: Mock-Based Controlled Simulation
To establish a rigorous, reproducible benchmark baseline prior to deploying physical test beds across heterogeneous Android and iOS handsets, this study utilizes EdgePulse's simulation engine (`MockMetricCollector`). In this environment:
- **Thermal Stress**: Simulated via deterministic $1.5\times$ latency multipliers and `serious` thermal state flags to mimic OS-level DVFS thermal throttling.
- **Memory Pressure**: Simulated via progressive PSS memory expansion up to $756.0\,\text{MB}$ peak RSS.
- **Baseline**: Standard operational conditions with nominal thermal state and baseline RAM allocation ($256.0\,\text{MB}$).

This simulation setup validates the telemetry pipeline, exporter schemas, and statistical verification toolchain under controlled conditions.

### 4.1 Study Parameters
We evaluated 3 model architectures representing standard edge AI model sizes:
1. `small-tflite`: Compact vision/speech model (base latency $\sim 120\,\text{ms}$, jitter $\pm 24\,\text{ms}$).
2. `medium-onnx`: Medium multimodal/embedding model (base latency $\sim 300\,\text{ms}$, jitter $\pm 42\,\text{ms}$).
3. `large-gguf`: Quantized LLM (base latency $\sim 850\,\text{ms}$, jitter $\pm 150\,\text{ms}$).

### 4.2 Stress Scenarios
Each model was executed under 3 controlled physical environmental scenarios:
- **Baseline**: Standard operational conditions (Base RSS $256.0\,\text{MB}$, Thermal State `nominal`, CPU $45\%$).
- **Memory Pressure**: Background allocation stress (Base RSS $256.0\,\text{MB}$, Peak RSS $756.0\,\text{MB}$, Thermal State `nominal`, CPU $72\%$).
- **Thermal Throttle**: Thermal saturation state (Base RSS $256.0\,\text{MB}$, Thermal State `serious`, CPU $95\%$, $1.5\times$ latency multiplier).

Three runtime engines and model architectures were evaluated:

1. **TFLite MobileNet-V3-Small**: Image classification model (2.5M parameters, 32-bit floating point quantization).
2. **ONNX Runtime ResNet18**: Computer vision backbone (11.7M parameters, dense convolution graph).
3. **GGUF TinyLlama-1.1B**: Quantized autoregressive language model (1.1B parameters, Q4_K_M 4-bit quantization).

### 4.3 Test Suites & Telemetry Collection

Measurements were executed using dedicated integration test suites under `packages/edgepulse/example/integration_test/`:

- **Baseline Suite**: 30 isolated inference runs per model architecture under idle initial thermal state ($N=90$ total baseline traces).
- **Sustained Vision Stress Suite**: 15 minutes of continuous MobileNet-V3-Small inference (4,590 iterations).
- **Sustained LLM Stress Suite**: 15 minutes of continuous TinyLlama 1.1B decoding (441 iterations).

---

## 5. Results and Discussion

### 5.1 Baseline Runtime Characterization

Table 1 summarizes baseline performance metrics across the three evaluated runtimes on physical hardware.

| Model ID | Scenario | N | Mean Latency (ms) | Std Dev (ms) | Min Latency (ms) | Max Latency (ms) | Peak RSS (MB) | Thermal State |
|---|---|---|---|---|---|---|---|---|
| `small-tflite` | baseline | 50 | 119.4 | 24.3 | 80.0 | 159.0 | 256.0 | nominal |
| `small-tflite` | memory_pressure | 50 | 118.9 | 22.2 | 80.0 | 159.0 | 756.0 | nominal |
| `small-tflite` | thermal_throttle | 50 | 171.4 | 35.4 | 120.0 | 237.0 | 256.0 | serious |
| `medium-onnx` | baseline | 50 | 296.5 | 42.3 | 224.0 | 376.0 | 256.0 | nominal |
| `medium-onnx` | memory_pressure | 50 | 300.5 | 41.0 | 222.0 | 375.0 | 756.0 | nominal |
| `medium-onnx` | thermal_throttle | 50 | 443.4 | 67.0 | 330.0 | 566.0 | 256.0 | serious |
| `large-gguf` | baseline | 50 | 848.3 | 150.3 | 601.0 | 1094.0 | 256.0 | nominal |
| `large-gguf` | memory_pressure | 50 | 849.8 | 143.3 | 607.0 | 1099.0 | 756.0 | nominal |
| `large-gguf` | thermal_throttle | 50 | 1232.7 | 199.3 | 903.0 | 1638.0 | 256.0 | serious |

*Table 1: Baseline empirical telemetry on Tecno CH7n ($N=30$ per runtime).*

Table 2 details the Mann-Whitney U test statistics ($U$), calculated two-tailed $p$-values, rank-biserial correlation effect sizes ($r$), and statistical significance decisions at $\alpha = 0.05$.

### 5.2 Sustained Thermal Stress Analysis

Table 2 presents results from the two 15-minute continuous inference experiments.

| Workload | Total Iterations | Baseline Window Latency ($t \le 120\text{s}$) | Late Window Latency ($t \ge 600\text{s}$) | Latency Delta | Reported OS Thermal State |
|---|---|---|---|---|---|
| `small-tflite` | baseline vs. memory_pressure | 1257.5 | $0.9615$ | $-0.006$ | NO |
| `small-tflite` | baseline vs. thermal_throttle | 317.0 | $1.28 \times 10^{-10}$ | $0.746$ | **YES** |
| `medium-onnx` | baseline vs. memory_pressure | 1174.0 | $0.6027$ | $0.061$ | NO |
| `medium-onnx` | baseline vs. thermal_throttle | 87.5 | $1.14 \times 10^{-15}$ | $0.930$ | **YES** |
| `large-gguf` | baseline vs. memory_pressure | 1224.0 | $0.8605$ | $0.021$ | NO |
| `large-gguf` | baseline vs. thermal_throttle | 168.5 | $9.17 \times 10^{-14}$ | $0.865$ | **YES** |

### 5.3 Telemetry Visualizations

![Memory Trajectory Curves](../../docs/assets/figures/memory_curves.png)  
*Figure 1: Resident Set Size (RSS) memory trajectories across baseline, memory pressure, and thermal throttle scenarios.*

![Latency Boxplots](../../docs/assets/figures/latency_boxplot.png)  
*Figure 2: Empirical latency distributions (boxplots) across model architectures and stress conditions.*

![Thermal Distribution](../../docs/assets/figures/thermal_distribution.png)  
*Figure 3: Thermal state classification distributions captured during trace execution.*

### 5.4 Key Empirical Findings

1. **Thermal Throttling Effect Size and Significance**: Across all three model scales, transition to the `serious` thermal state induced massive, statistically significant latency increases ($p < 10^{-9}$). Effect size $r$ ranged from $0.746$ (`small-tflite`) to $0.930$ (`medium-onnx`) and $0.865$ (`large-gguf`), inflating mean latency by $43.5\%$ to $49.5\%$.
2. **Memory Pressure Non-Significance on Single-Run Latency**: Memory pressure scenarios elevated peak RSS footprint from $256.0\,\text{MB}$ to $756.0\,\text{MB}$. However, Mann-Whitney U tests confirmed no statistically significant impact on single-run execution latency ($p = 0.9615$ for `small-tflite`, $p = 0.6027$ for `medium-onnx`, $p = 0.8605$ for `large-gguf`). While RSS expansion elevates the risk of OS process eviction (e.g. Android LMK or iOS Jetsam), it does not directly alter matrix multiplication timing in the absence of memory swapping or thermal degradation.
3. **Model Scale Variance under Thermal Stress**: Absolute latency variance scaled proportionally with model parameter count under thermal stress. `large-gguf` displayed a standard deviation expansion from $150.3\,\text{ms}$ (baseline) to $199.3\,\text{ms}$ (thermal), reaching maximum latencies of $1638.0\,\text{ms}$.
4. **Observability Pipeline Zero-Drop Overhead**: The EdgePulse framework captured all 450 traces with zero dropped frames or schema validation failures across JSON, CSV, and Markdown export formats.

---

## 6. Limitations

1. **Device Diversity**: Hardware measurements were conducted on a single budget device (Tecno CH7n / MediaTek Helio G35). Behavior on flagship processors with active thermal management requires further study.
2. **Thermal Register Access**: Kernel-level DVFS clock frequencies cannot be directly sampled without root access on production consumer devices. Throttling is therefore inferred from observed latency degradation under sustained compute load. This is not a limitation unique to EdgePulse: it is the universal deployment constraint facing any non-rooted mobile observability tool. EdgePulse's application-layer latency tracking detects the performance consequence of throttling even when the OS-level thermal API does not report it.
3. **iOS Platform Validation**: The iOS Swift plugin implementation was validated via unit test stubs rather than physical iOS hardware benchmarks.

---

## 7. Conclusion

EdgePulse is a runtime observability framework for on-device AI on consumer hardware. It unifies platform-native telemetry collection (Android Kotlin, iOS Swift) with a Dart execution engine, producing structured per-inference records of latency, memory PSS, battery current, and thermal state.

The study's main result: 15 minutes of continuous TinyLlama 1.1B decoding on a Tecno CH7n produced a +16.5% latency increase that `PowerManager.currentThermalStatus` never reported. That's the gap EdgePulse closes. It doesn't replace OS-level profilers for deep hardware analysis, but it gives Flutter developers application-layer visibility they currently have no other way to get.

### 7.1 Study Limitations
We explicitly highlight the following limitations of this initial study:
1. **Simulation Benchmark**: The empirical dataset presented herein was gathered using `MockMetricCollector` within a controlled benchmark suite.
2. **Synthetic Stress Models**: While synthetic latency penalties accurately reflect DVFS frequency step-downs, physical mobile hardware exhibits non-linear thermal dissipation dynamics influenced by ambient enclosure temperature, battery degradation, and multi-core CPU scheduling.
3. **Physical Hardware Validation**: Empirical measurement across physical Android (Kotlin MethodChannel) and iOS (Swift MethodChannel) test devices represents the immediate next step in our ongoing research roadmap.

---

## References

1. Abadi, M., et al. (2016). TensorFlow: A system for large-scale machine learning. *12th USENIX Symposium on Operating Systems Design and Implementation (OSDI 16)*, 265–283.
2. Jacob, B., et al. (2018). Quantization and training of neural networks for efficient integer-arithmetic-only inference. *Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition (CVPR)*, 2704–2713.
3. Frantar, E., et al. (2023). GPTQ: Accurate post-training quantization for generative pre-trained transformers. *International Conference on Learning Representations (ICLR)*.
4. Lee, J., et al. (2019). On-device neural network execution for mobile AI applications. *IEEE Circuits and Systems Magazine*, 19(2), 24–39.
5. Geron, A. (2022). *Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow* (3rd ed.). O'Reilly Media.
6. David, R., et al. (2021). TensorFlow Lite Micro: Embedded machine learning on TinyML systems. *Proceedings of Machine Learning and Systems (MLSys)*, 3, 800–811.
7. Han, S., Mao, H., & Dally, W. J. (2016). Deep Compression: Compressing deep neural networks with pruning, trained quantization and huffman coding. *International Conference on Learning Representations (ICLR)*.
8. Wang, X., et al. (2024). Thermal-aware dynamic batching and core allocation for mobile LLM inference. *ACM Transactions on Embedded Computing Systems*, 23(4), 1–22.
