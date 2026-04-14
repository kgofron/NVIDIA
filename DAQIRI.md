# DAQIRI

| | |
|--|--|
| **Version** | 1.0 |
| **Author** | K. Gofron |
| **Date** | 4/14/2026 |
| **Last modified** | 4/14/2026 |

That phrase refers to a **data acquisition and processing capability**—specifically, a system designed to **move data efficiently from detectors (sensors) to GPUs for real-time processing**.

### What it means in practical terms

* **“Sensors”** = detectors, instruments, or DAQ hardware generating raw data (e.g., neutron detectors, cameras, digitizers)
* **“GPUs”** = high-performance processors used for fast computation (e.g., image reconstruction, filtering, AI/ML, event processing)
* **“Streamlines connectivity”** = reduces latency, overhead, and complexity in getting data from the hardware into GPU memory

---

## What DAQIRI likely does (conceptually)

A tool like *DAQIRI* typically provides:

* **High-throughput data pipelines**
  Efficient transfer of large data streams (often GB/s) from DAQ systems to compute nodes

* **Low-latency transport**
  Uses optimized networking (e.g., RDMA, zero-copy, shared memory) to avoid CPU bottlenecks

* **Direct GPU ingestion**
  Moves data **directly into GPU memory**, bypassing unnecessary copies through system RAM

* **Real-time processing enablement**
  Allows on-the-fly:

  * event filtering
  * image reconstruction
  * feature extraction
  * AI/ML inference

* **Simplified integration layer**
  Provides APIs or middleware so developers don’t have to manually manage:

  * network transport
  * buffer management
  * synchronization between DAQ and compute

---

## NVIDIA positioning (early preview, Holoscan)

NVIDIA describes **DAQIRI** as connecting **high-bandwidth sensors to GPU**—**plug-and-play** framing: *any sensor, any domain* with **automatic batching to GPU tensors**. In their materials it is a **subcomponent of Holoscan**; they intend to **productize** it and have offered a **pre-release** version for evaluation.

**Context from discussions:** conversations with NVIDIA include a recent discussion (e.g., with YQ) and earlier meetings with others on **leveraging Holoscan for Timepix** data. The open thread is how to use this capability in our environment, not only what the tool does in the abstract.

### How they contrast “before” vs “with” DAQIRI

* **Without DAQIRI:** streaming Ethernet to the GPU often means **DPDK**-style bring-up—**esoteric** network programming, large code volume (they cite order-of-**hundreds of lines** even for basic header/data split), limited uptake of **CUDA-X** in real-time high-bandwidth sensor apps, and friction when **switching** between data-movement libraries.
* **With DAQIRI:** a **small API surface** (e.g., init, load **YAML** config, **receive burst** into GPU tensor batches) plus **human-readable YAML** for manager type (e.g. DPDK), CPU/GPU memory regions (buffer sizes, huge pages), NIC PCIe address, RX queues, flow control, etc. They cite **~100 Gbps per NIC/GPU** class targets and **DMA** from the NIC into **GPU tensors**, with a **modular** architecture intended to track **current and future** network stack libraries.

---

## Why this matters (especially in your domain)

For systems like **EPICS / areaDetector / ADARA pipelines**:

* Traditional flow:
  **Detector → CPU memory → disk → post-processing**

* With tools like DAQIRI:
  **Detector → GPU (near real-time) → immediate analysis**

### Benefits

* Faster feedback during experiments
* Reduced data backlog and storage load
* Enables **online analysis** instead of offline workflows
* Better utilization of modern GPU-based compute clusters

---

## Evaluation: preview risk vs upside

**Upside**

* **Lower integration tax** for Ethernet → GPU at high rate; CUDA-X and custom signal-processing stacks become reachable without owning full DPDK bring-up.
* **Holoscan alignment** if we want a single ecosystem from ingest through GPU operators later.

**Risks / questions to resolve with NVIDIA**

* **Pre-release churn:** API, packaging, licensing, and support until productization.
* **Timepix3 / Timepix4 fit:** framing (**UDP / custom headers**), **variable event size**, **multi-stream** and triggers—does their **header/data split + batching** model match our wire format (and readout: **SERVAL** vs **SPIDR4** / **Katherine** / vendor), or do we need a shim?
* **Holoscan scope:** minimum stack on **DGX**—DAQIRI-only vs full Holoscan app skeleton.
* **Hardware path:** assumed **NIC**, **GPUDirect / RDMA** vs **DPDK-only** staging—drives **latency**, **CPU load**, and **timestamping** story for science-grade alignment.

---

## Timepix3 / Timepix4 fit and alternatives

### When DAQIRI is a good fit

DAQIRI is aimed at **high-rate streaming Ethernet** into a **host NIC**, with **batched** handoff toward **GPU memory** (DPDK-class ingest, not “any TCP application”). It is a **good candidate** for Timepix-family work **only when** the deployment actually has such a **packet path** and **framing** that the preview can consume (or that you can supply via **replay**, a **parallel Ethernet leg**, or a **thin bridge** from an existing stack).

### Timepix3

* **Typical facility / EPICS path:** readout board → **SERVAL** → **HTTP/JSON control** and **TCP**-based **64-bit TPX3-style** streams to clients; **ADTimePix3** is a **SERVAL client** on that path (see [ASI stack / bit-width pipeline](https://github.com/areaDetector/ADTimePix3/blob/master/documentation/TimePix3_pipeline_48_64_96_caption.png)). That is the **supported integration** for areaDetector, not DAQIRI.
* **Mismatch to watch:** **TCP streams from SERVAL** are not the same problem as **raw Ethernet bursts** for DPDK RX. DAQIRI is **not automatically** the best **default** for Timepix3 if the primary goal is **standard EPICS**—**SERVAL + ADTimePix3** already defines that stack.
* **Where DAQIRI still makes sense:** **NIC- and GPU-bound** online processing, **replay** of captured traffic, a **firmware/readout Ethernet** path that matches DPDK-style ingest, or a **shim** that republishes the same logical events in a **UDP / custom** framing DAQIRI can ingest.

**Entry-point reminder (logical vs wire):** ingest tools attach to **what hits the NIC** (often originating at the **readout board**), not the **Timepix3 chip** itself. The **documented app entry** for ADTimePix3 is **SERVAL**; mapping SERVAL to DAQIRI is a **transport and packaging** decision, not “closer to the ASIC.” The optional **LUNA** **96-bit** path is **parallel** to the ADTimePix3 path and is **not** used by ADTimePix3.

### Timepix4

* Timepix4 pushes **much higher sustained throughput** than typical Timepix3 lab setups; common readout systems (e.g. **SPIDR4**-class stacks) often use **10GbE** (and chip-level link budgets are much larger than Timepix3). That moves more use cases into the **bandwidth regime** where DAQIRI’s story (**high Gbps, batch to GPU tensors**) is **architecturally** plausible.
* DAQIRI is still **not “Timepix4-aware”** by itself—it is **transport + GPU staging**. Fit depends on the **actual** readout (**Katherine**, **SPIDR4**, vendor **Kepler**-class software, etc.), **wire format**, and **control model**. Productized or preview DAQIRI must be validated against **that** path, not the ASIC name alone.

### “Better” technology depends on goals

There is **no single winner**; choice depends on **target Gbps**, **who owns the readout**, whether **EPICS stays in the hot path**, and **custom engineering** tolerance.

| Priority | Often stronger fit than DAQIRI for that job |
|----------|---------------------------------------------|
| **EPICS / facility-standard DAQ** | **Vendor/readout stack + areaDetector** (e.g. **ADTimePix3** + **SERVAL** for Timepix3). |
| **Moderate rates; GPU analytics** | **CPU-side ingest** (driver / existing buffers) then **CUDA**, **PyTorch**, **RAPIDS**; add **pinned / zero-copy** where profiling shows need—simpler if the NIC is not the bottleneck. |
| **Maximum throughput; full control** | **Custom FPGA / readout** + **RDMA / GPUDirect** (or a hand-owned **DPDK** stack). Higher ceiling, higher build and maintenance cost. |
| **GPU streaming + application framework** | **Holoscan** (and related NVIDIA I/O pieces)—**with or without** DAQIRI; DAQIRI is one possible **ingest shortcut**, not the only door. |
| **Neutron / ADARA-style pipelines** | Facility patterns (**ADARA**, staging, clusters) unless there is a clear **online GPU** requirement. |

### Short verdict

* **Timepix3:** For **normal controls, imaging, and EPICS**, prefer **SERVAL + ADTimePix3** unless you have a concrete **Ethernet packet** path and a **GPU/NIC-bound** requirement; treat DAQIRI as a **targeted experiment** (often **replay-first**).
* **Timepix4:** **More often** in the performance class DAQIRI targets, but **vendor stack alignment** and **framing** still decide; alternatives (**FPGA/RDMA**, vendor-native paths) may win on efficiency at the cost of bespoke work.

---

## Proposed path: Timepix3 → DAQIRI → DGX (replay first)

A practical sequence:

| Phase | What you prove | Why it helps |
|--------|----------------|--------------|
| **Replay** | Same **packet shapes**, **rates**, and **batch boundaries** as production; tensors resident on GPU and downstream kernels (sort, histogram, clustering, ML) | Decouples from beam/live DAQ; **repeatable** benchmarks when the preview updates |
| **Live** | Loss, backpressure, clocks, orchestration / EPICS hooks | After replay shows **sustained rate** and **correct semantics** |

**Replay details to decide:** where the replay source runs (same host as DGX vs a **replay appliance** feeding the DGX NIC), **timestamp preservation**, and whether later live use needs **NIC hardware timestamping**.

---

## Suggested meeting outcomes

1. **Access:** preview delivery (build, docs, examples), **NDA / export** if applicable.
2. **Reference path:** minimal **Timepix-like UDP → DAQIRI → tensor → CUDA** sample or closest official demo.
3. **Timepix3 / Timepix4 mapping:** wire format, expected **Gbps**, readout stack (**SERVAL** vs **SPIDR4** / **Katherine** / other), single vs multi-detector, triggers—recommended **YAML** pattern.
4. **Holoscan:** whether **DAQIRI-only** on DGX is supported for the first milestone vs requiring a Holoscan app wrapper.
5. **Success criteria:** sustained rate, copy count, CPU utilization, latency for a named workload.
6. **Next step:** **replay artifact** (e.g. pcap or raw stream), owner, **timeline**, target **DGX** configuration (NIC, driver stack).

---

## One-line interpretation

> *DAQIRI is a high-performance data pipeline layer that enables real-time, low-latency transfer of detector data directly to GPUs for immediate processing.*

---

## Paste-ready summary (meeting / email)

*NVIDIA is offering an early preview of DAQIRI, a Holoscan-related component that streamlines high-bandwidth Ethernet sensor ingest into GPU memory (replacing large DPDK-style bring-up with a small API plus YAML configuration and batched receive into GPU tensors). Following discussions with NVIDIA—including recent conversation with YQ and earlier Holoscan/Timepix discussions with others—we should evaluate fit for our **Timepix3** workflows on the DGX, and keep **Timepix4** (higher-rate readouts such as SPIDR4-class paths) in mind for the same transport questions. Pre-release access is attractive but carries API and support uncertainty until productization; the meeting should clarify packaging, Holoscan dependencies, and framing assumptions relative to **SERVAL/TCP** (typical ADTimePix3 path) vs **raw Ethernet** ingest. For EPICS-centric Timepix3 use, **SERVAL + ADTimePix3** remains the default stack; DAQIRI is most compelling when we have a concrete **NIC packet** path or **replay**. A practical first step is to route Timepix3 traffic through DAQIRI on the DGX beginning with **replayed** captures to validate throughput, batching, and GPU-side processing before coupling to live acquisition.*
