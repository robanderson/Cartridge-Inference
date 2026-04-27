# Cartridge Inference: Cloud-Prefill + Client-Decode with Pre-Shared Weights for Hyper-Scalable Frontier LLMs

**Author:** Rob Anderson
**Date:** 27 April 2026
**GitHub:** https://github.com/robanderson/Cartridge-Inference.git

## Author Bio

Rob Anderson is an ex-IBM Mainframe Assembler programmer who worked with Databank NZ and EDS in the 1990s. His early career focused on large-scale virtual memory systems, demand paging, and predictive prefetching on IBM S/390 MVS platforms. The same core principles — a small fast working set, a large backing store, and a predictive pager that hides latency behind compute — translate directly to the problem of running trillion-parameter Mixture-of-Experts (MoE) models on client hardware. Cartridge Inference is the result of that translation.

## Abstract

Frontier LLM providers are losing substantial sums on high-usage "Max" and "Unlimited" plans because decode (autoregressive token generation) dominates cost and is memory-bandwidth bound.

**Cartridge Inference** is a vision for a new distribution model that addresses this:

- Providers ship **pre-shared weights** on an encrypted, **read-only** Thunderbolt/USB-C cartridge — fixed-content hardware in the spirit of a console game cartridge, not a writable USB drive.
- Cloud performs only prefill, expert-routing prediction, and a single batched forward pass per client-generated chunk.
- Client hardware runs the heavy decode phase using paged MoE experts loaded from the local cartridge.
- 10–32 token chunks are periodically synced back to the cloud to maintain perfect KV-cache state.

By chunk arithmetic alone, the cloud performs roughly one forward pass per chunk instead of one per token. With 32-token chunks that is approximately a 32× reduction in cloud decode work, which the paper rounds to **~20–28×** after sync, hint generation, and overhead. The result is dramatically lower cloud decode cost, near-local latency for users, strong IP protection, and horizontal scaling by adding more client nodes and cartridges.

This is a vision document, not an engineering specification. No model has been built or benchmarked. The numbers below are derived from straightforward arithmetic on chunk size; real-world multipliers will be established by implementation.

## 1. Introduction & Motivation

Current inference economics are unsustainable. Heavy users on Max plans generate millions of output tokens per month, with decode responsible for the vast majority of compute spend. Providers need a way to serve frontier models profitably while giving power users better performance. Cartridge Inference proposes a distribution architecture that shifts the memory-bandwidth-bound work to client hardware that already has the right characteristics, while keeping the provider in authoritative control of context, safety, and model IP.

## 2. Core Architecture — Pre-Shared Weights + Client Decode

- **Pre-shared weights** reside on the user's encrypted cartridge (read-only, hardware-protected).
- The cloud runs prefill and predicts the active MoE experts for the upcoming chunk.
- Only the KV cache and compact routing metadata are streamed to the client.
- The client loads only the needed experts from the fast local cartridge into unified memory.
- Decode runs locally using a virtual, paged expert system: demand paging plus predictive prefetch.

### 2.1 The Cartridge: Read-Only by Design

The cartridge is not a writable USB drive. It is a read-only artefact, in the spirit of an NES/SNES/N64 game cartridge — fixed content, immutable after manufacture. This has three consequences:

- **Stronger IP protection.** No writable storage means no firmware-update attack surface and a smaller secure-boot trust boundary.
- **Cleaner lifecycle.** A new model is literally a new cartridge, mailed to the user. There is no over-the-air weight refresh to attack.
- **Cultural framing.** "Cartridge" signals a discrete, owned, version-locked artefact. It is a more memorable and architecturally honest metaphor than "encrypted USB stick."

### 2.2 Client-Side Expert Pager (S/390 Virtual Memory Analogy)

The decode engine behaves as if the entire model is in fast memory. A lightweight runtime — inspired by classic MVS pageable memory management — pages experts in and out from the cartridge backing store, using cloud-provided prefetch hints. This allows trillion-parameter MoE models to run efficiently on 128 GB-class client devices.

### 2.3 Learned Expert Prefetcher

Expert activation in MoE models is strongly subject-matter dependent: a coding conversation activates a different expert subset than a French-language conversation or a mathematical proof, and those subsets evolve slowly within a session.

Cartridge Inference proposes a small auxiliary **learned predictor model** running on the client. Its inputs are:

- The most recent 32-token chunk.
- The set of experts that activated for that chunk.

Its output is a probability matrix over experts likely to be needed for the next 32 tokens. The pager uses that matrix to:

- Page **in** any high-probability experts not currently resident.
- Page **out** experts whose usage probability has decayed below a threshold (working-set eviction, classic LRU/LFU style).

This is a learned prefetcher — the modern equivalent of the reference-bit page-replacement logic used in MVS — with the twist that it predicts on semantic locality, not just temporal recency. Prefetch accuracy is an open research question and is acknowledged here as the central engineering challenge of Cartridge Inference, in the same way that TLB miss rates were the central challenge of early virtual memory.

## 3. Chunked Sync with Cloud Re-Synchronisation

- The client decodes autonomous chunks of **10–32 tokens**.
- The client sends the tiny token-ID payload back to the cloud.
- The cloud runs **one batched forward pass over the chunk** — prefill-style work, parallel across all 32 positions — and uses it to extend its authoritative KV cache.
- Because the same weights run on both ends, the cloud trusts the client's tokens as ground truth and simply computes the canonical KV-cache entries for them.

The cost insight: **decode is sequential and memory-bandwidth bound** (each token requires reading the full model weights from HBM), whereas **prefill over a known sequence is parallel and compute-bound** (weights read once, K/V computed for many positions at once). Replacing 32 sequential decode steps with one batched 32-token pass yields, by simple arithmetic, ~32× less cloud decode work. The headline figure of ~20–28× allows for sync, hint generation, and overhead.

## 4. KV-Cache Transport

Cartridge Inference is targeted at users on **300 Mbps+ fibre or Starlink-class connections**. KV-cache handoff is one-time per turn and benefits from two complementary techniques:

- **KV-cache quantisation** — DeepSeek-style Multi-head Latent Attention (MLA) and similar techniques compress the KV state by an order of magnitude with negligible quality loss.
- **Incremental KV sync** — within a multi-turn conversation, only the *delta* KV (new tokens since the last sync) is transferred. Prior KV chunks remain cached client-side, so a 200k-token context built up over many turns never re-pays its full transfer cost.

These two techniques together keep KV transport well within the bandwidth budget of a modern fibre link, while preserving the "near-local latency" property of decode itself.

## 5. Hardware Affinity

Decode is memory-bandwidth bound, not compute-bound. Apple's M5 series — particularly M5 Max and (when released) M5 Ultra, with 128 GB+ unified memory and very high bandwidth — is a strong fit. Adding more M5 nodes plus cartridges scales tokens-per-second near-linearly via **n+1 local routing** (load balancing plus redundancy). Equivalent NVIDIA workstation hardware (RTX 6000 Ada / A6000-class) is also a viable target.

## 6. Commercial Model — Illustrative Example

The figures below are **illustrative only**. Real-world pricing will be set by each provider based on implementation cost and the perceived value of the system to the customer.

**Example: "Claude 4.8 Max+ Team Edition"**

- **Cartridge:** illustrative one-time hardware purchase per seat (encrypted Thunderbolt 5 read-only cartridge).
- **Subscription:** illustrative per-user or team flat rate.
- **Minimum client spec:** 128 GB unified-memory system (M5-class Mac or equivalent RTX 6000 Ada / A6000 workstation).

**Benefits**

- Dramatically higher (or effectively unlimited) usage limits.
- Near-local decode latency.
- Decode performed on customer hardware, sharply reducing provider cloud cost.
- Users self-scale performance by adding more nodes and cartridges.

### 6.1 Lifecycle and Replacement

Because the cartridge is read-only, model updates ship as new physical cartridges. A new frontier model release means a new cartridge mailed to the subscriber. Providers may choose to bundle upgrades into the subscription or to charge for them separately; that choice is a commercial decision, not an architectural one. Lost, stolen, or failed cartridges are revoked cloud-side via key invalidation, and a replacement is shipped under whatever policy the provider sets.

### 6.2 Provider-Neutral Framework

Cartridge Inference is not specific to any single provider. The same architecture is equally available to any frontier LLM provider — including non-US providers such as Moonshot (Kimi), Zhipu (GLM), Alibaba (Qwen), DeepSeek, Mistral, and others. Each provider inherits its own jurisdiction's regulatory regime; the architecture itself is jurisdiction-neutral.

## 7. User Experience & Scaling

Marketed as **"Bring Your Own Decode"**:

> "Want faster tokens per second? Add more local compute nodes and cartridges. Your hardware becomes part of the distributed decode pool."

The system uses **n+1 local routing** so performance scales cleanly and remains reliable as users grow their private cluster.

## 8. Security & IP Protection

Cartridge Inference is designed to make weight extraction more expensive than simply paying for the subscription. It does not aim to be unbreakable against a well-funded nation-state adversary with a decap lab — no consumer hardware DRM scheme survives that threat — but it does aim to be commercially robust.

The defence is layered:

- **Encryption at rest.** Weights are encrypted on the read-only cartridge.
- **Hardware enclave decryption.** Decryption keys live inside a hardware root-of-trust — Apple Secure Enclave / Microsoft Pluton / Google Titan-class — and weights are decrypted only inside that boundary.
- **Memory encryption at use.** Decrypted expert pages live in encrypted-memory regions of the host SoC (AMD SEV-SNP / Intel TDX / Apple's equivalent), inaccessible to the OS or DMA.
- **Cloud guardian layer.** Final safety, policy, and watermarking decisions remain cloud-side.

### 8.1 Open Research Direction — Partial Model on Cartridge

A stronger property — and an acknowledged area of further work — is to ensure the cartridge alone does **not** contain a fully runnable model. Possible directions include keeping certain layers cloud-resident, retaining a small set of "crown-jewel" experts cloud-side, holding the embedding/unembedding projection in the cloud, or issuing per-session keys for a subset of layers. In all such designs, full extraction of the cartridge yields an incomplete artefact, and the cloud remains a necessary participant. This is flagged honestly as a known weakness needing further work.

## 9. Failure Modes & Graceful Degradation

Cartridge Inference is designed to never be *worse* than today's pure-cloud baseline:

- **Cartridge disconnected, lost, or failed.** The client transparently routes the session to pure-cloud inference, exactly as today. The provider absorbs the additional decode cost during this fallback window. The user sees no service interruption beyond standard cloud latency.
- **Cloud unreachable.** The session pauses until reconnection. Under the §8.1 partial-model design, the client alone is intentionally insufficient to generate, which is both a security property and a clear failure semantics.
- **Both available (normal mode).** Full Cartridge Inference economics apply.

This symmetry — neither half useful without the other in normal operation, but graceful fallback to today's status quo when the cartridge is absent — is a deliberate design property.

## 10. Comparison with Existing Approaches

| Approach | Frontier-capable? | Weight IP protected? | Decode cost borne by | User latency | Horizontal scale by user |
|---|---|---|---|---|---|
| Pure cloud API (today's default) | Yes | Yes | Provider | Network-bound | No |
| Pure local inference (Ollama, LM Studio, on-device) | No (small/open models only) | N/A | User | Local | No |
| Open-weight downloads (Llama, Mistral, Kimi, GLM) | Partial | No | User | Local | No |
| Speculative decoding (Medusa, EAGLE) | Yes | Yes | Provider | Network-bound | No |
| Confidential compute (H100 CC, AMD SEV) | Yes | Yes (in cloud) | Provider | Network-bound | No |
| Apple Private Cloud Compute | Limited | Yes | Provider | Network-bound | No |
| **Cartridge Inference** | **Yes** | **Yes** | **User (decode) + Provider (prefill)** | **Near-local** | **Yes (more nodes + cartridges)** |

Cartridge Inference is, to the author's knowledge, the only architecture that simultaneously keeps frontier weights protected, shifts decode cost to client hardware, retains cloud authority over context and safety, gives the user near-local latency, and lets the user scale their own throughput by adding hardware.

## 11. Economic Impact

Providers reduce marginal decode cost substantially while delivering superior performance to customers who already own capable hardware. This is a realistic path to hyper-scale frontier AI without unlimited GPU spend. The exact economics will be borne out by implementation and by the value users perceive in the system.

## Conclusion

Cartridge Inference — pre-shared weights on a read-only cartridge, paged MoE experts, a learned expert prefetcher, and chunked cloud sync via prefill-style re-synchronisation — offers a practical, economically sustainable way to serve frontier models at massive scale. It leverages existing high-end client hardware, restores provider profitability on premium plans, and gives users faster, more private inference. The remaining engineering challenges — prefetch accuracy, partial-model design, and KV-cache transport optimisation — are concrete, addressable, and worth pursuing.

**Published:** 27 April 2026
**Author:** Rob Anderson (ex-IBM Mainframe Assembler, Databank NZ & EDS)
