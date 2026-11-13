# Open questions and things to tweak

This document collects the decisions that are not yet made, with the numbers that constrain them. Where a default is proposed it is marked as such. The rest of the documentation describes the baseline; this document is where it is allowed to be wrong.

## 1. Model dimensions: what checks out

With `d = 2048`, 8 heads and 512 per head:

- `8 × 512 = 4096 = 2 × 2048`, so the concatenated heads and the gate `SiLU(X W_g)` have the same width and the Hadamard product is well formed.
- Per layer: `W_g`, `W_q`, `W_k`, `W_v` (2048 × 4096 each) and `W_o` (4096 × 2048), about 41.9M parameters.
- Encoder: 3072 (image) + 2048 (audio) + 2048 (last embedding) = 7168 in, 2048 out, about 14.7M parameters.

The number of CC layers `N` is not yet chosen:

| N   | CC parameters | Weights, int8 | Weights, fp16 |
| --- | ------------- | ------------- | ------------- |
| 16  | 0.67B         | 0.67 GB       | 1.3 GB        |
| 32  | 1.34B         | 1.3 GB        | 2.7 GB        |
| 48  | 2.01B         | 2.0 GB        | 4.0 GB        |
| 64  | 2.68B         | 2.7 GB        | 5.4 GB        |
| 96  | 4.03B         | 4.0 GB        | 8.1 GB        |

**Proposed default:** 48 layers at int8, about 2 GB of weights, leaving room for the context state on an 8 to 12 GB phone. Revisit once the KV cache question below is settled, since that dominates memory.

Small corrections to the written equation:

- The value projection should apply to the context tokens: `X_j W_v`, not `X_i W_v`.
- The equation has no residual connection and no normalization. Both are almost certainly needed for a deep chain with learned routing (a layer that may be applied repeatedly to the same token needs its output to stay in the same distribution as its input). Proposed: pre-norm (RMSNorm) and a residual, `X'_i = X_i + Layer(Norm(X_i))`.
- No positional encoding is mentioned. With a sliding window, rotary embeddings applied to cache-relative positions is the standard StreamingLLM choice.

## 2. The KV cache is the real memory problem

Each layer needs keys and values for every token it can attend to. With 4096-wide K and 4096-wide V per token, one layer's cache over a 32k window is **512 MB at fp16, 256 MB at int8**. If every layer keeps its own cache (as in a normal transformer), that is 16 GB at int8 for 64 layers. A phone cannot hold that.

The routed design makes the question subtler than in a normal transformer, because layers do not run in a fixed order and each token is only ever "enriched" by whichever layers the router chose for it. Options:

| Option | Memory (32k window, int8) | Trade-off |
| --- | --- | --- |
| A. Per-layer caches of the tokens each layer processed | Up to 256 MB × N, in practice less because each token visits few layers | Simplest semantics, worst case unaffordable, and a layer cannot read tokens it never processed. |
| B. One shared K/V stream: single `W_k`, `W_v` shared by all layers, computed once when a token is written | 256 MB total | Layers keep their own `W_g`, `W_q`, `W_o`. The tape is literally one tape that every layer reads. Fits the Turing-machine framing. Proposed default. |
| C. Grouped K/V heads (fewer K/V heads than query heads) on top of A or B | B with 2 KV heads: 64 MB | Standard trick, composes with B. |
| D. Shorter window | Linear in window | Costs memory horizon (see 4). |

**Proposed default:** B, optionally with C. In B the "write" to the tape is a single K/V pair per token, which also makes the per-cycle cost independent of which layer the router picked. Note that with B a token's K/V are fixed when it is written; the refinements the CC makes to the last token while it is still "under the head" need a rule for when the K/V for that token are (re)computed. Proposed: recompute the last token's K/V after every cycle, freeze when the next token is written.

## 3. Inference cadence vs. audio sample rate

"Match the inference frequency to the audio sample rate" has to mean the audio _frame_ rate, not the sample rate. One cycle on one layer costs about 0.3 GMAC of compute and, more importantly, reads the whole KV cache. At a literal 16 kHz cadence that is about 10 TFLOP/s and over 4 TB/s of memory traffic, far beyond any phone.

With a frame per cycle, the cadence follows from sample rate and frame size:

| Sample rate | Frame | Cycles/s | ms/cycle | Window covers | KV read, int8 | Compute      |
| ----------- | ----- | -------- | -------- | ------------- | ------------- | ------------ |
| 16 kHz      | 1024  | 15.6     | 64       | 35 min        | 4.2 GB/s      | 9.7 GFLOP/s  |
| 16 kHz      | 2048  | 7.8      | 128      | 70 min        | 2.1 GB/s      | 4.8 GFLOP/s  |
| 24 kHz      | 1024  | 23.4     | 43       | 23 min        | 6.3 GB/s      | 14.5 GFLOP/s |
| 48 kHz      | 1024  | 46.9     | 21       | 12 min        | 12.6 GB/s     | 29 GFLOP/s   |

Compute is negligible for an NPU at every row; memory bandwidth and power are what decide. Two things pull in opposite directions: a faster cadence gives snappier reactions and more "thinking cycles" per second of speech, but shrinks the time the window covers and raises steady power draw.

**Proposed default:** 16 kHz audio, 1024-sample frames, about 16 cycles per second, 64 ms per cycle. A reply that needs 10 layers of thought lands in under a second. Reconsider 24 kHz / 1024 if speech quality through the articulatory synthesizer wants the bandwidth.

A related question: should the audio input be raw samples (1024 numbers per frame) or a fixed, non-parametric feature transform (e.g. a mel spectrogram of the frame)? The latter is cheaper for the encoder to use and symmetric with foveation being a fixed transform on the image side. Proposed: fixed features, same dimension.

## 4. The window is short-term memory; what is long-term memory?

At 16 cycles per second, 32k tokens is about 35 minutes. The model is never reset, so anything that happened more than half an hour ago is gone from the tape, including the user's name, preferences and ongoing tasks. The model needs memory beyond the window. The Turing-machine framing suggests the shape of the answer: give the tape a second segment that persists.

Options, not mutually exclusive:

- **Readable store.** A retrieval mechanism where the decoder can emit a query and the encoder ingests the result as a token. Memory becomes another sense.
- **Writable store.** A decoder action that writes the current embedding (or a compressed summary) to persistent storage with a key. Memory becomes another action.
- **Attention sinks as learned state.** Keep more than a handful of sink tokens and let the model choose which tokens to promote to permanent status.

**Proposed:** readable plus writable store as decoder actions, designed so the stored items are embeddings the encoder can project. This has to be in the training setup, so it needs to be decided before training, not after.

## 5. Foveation and the "where am I looking" problem

- "1024 pixels, one per color channel" is read here as 1024 samples per channel, 3072 values. If it was meant as 1024 values total, the encoder input shrinks to 5120 and the image gets much coarser. Confirm.
- The model chooses the fixation point but does not otherwise know it. It should get the current fixation (and rotation) back as an input so that "look 200 px left" is a learnable relative move. Proposed: add fixation and rotation, normalized, to the encoder input.
- Rotation: is it the device orientation (landscape to portrait, decided by the OS) or a model-chosen rotation of the fovea? The former is a fixed pre-processing step; the latter is an action. Proposed: orientation is OS-handled, rotation is not a model action in the first version.
- The foveation sampling pattern (log-polar, concentric rings, something else) is a free choice. It is non-parametric so it can be changed without retraining only if it is changed before training.

## 6. Output vocabulary of the decoder

The decoder emits actions every cycle. Speech (articulator parameters) is a continuous, per-cycle output and fits naturally. The others are discrete and structured, which raises the question of how a model that produces one embedding per cycle emits a widget description or a URL:

- **Streaming tokens.** The decoder has a text head that emits one token of a structured language (widget description, URL, typed text) per cycle, and a "mode" signal that says whether the text head is active. A widget of 200 tokens takes 200 cycles (about 12 seconds at 16 Hz) to emit. Probably too slow for widgets, fine for typing.
- **Parallel decode.** When the mode signal says "widget", a separate lightweight decoder runs over the last embedding for several steps within one cycle. Keeps the cadence but adds a second loop.
- **Template plus slots.** The widget vocabulary is a fixed set of templates with slots; the decoder picks a template and fills slots over a few cycles. Fast, limited range.

**Proposed:** streaming tokens for typing and navigation, parallel decode for widgets. The widget description language should be designed together with the compositor team.

## 7. What exactly is in the window

The baseline says the encoder's output is appended as a new token and the last enriched embedding is fed back into the encoder. Two readings exist:

- The previous token stays in the window and a new token is appended (the window grows by one per encoder call).
- The previous token is replaced by the new one (the window grows only by the number of encoder calls, but the refinement history is lost).

Proposed: the first. Every encoder call writes a new cell on the tape. The cycles between encoder calls refine the last cell in place.

## 8. Decoder routing details

- Argmax or sampled routing? Argmax is deterministic and cheap; sampling helps training. Proposed: trained with sampling, run with argmax.
- Is there a "halt / idle" option that skips the CC layer entirely? When nothing is happening the model should be able to do close to nothing and save power. Proposed: yes, an explicit idle choice that runs only the decoder on the next cycle.
- Can the router pick a layer that has not been visited for a very long time? Yes by construction; this is the point of routing.

## 9. Kernel-side questions that affect the model

- NPU driver interface: command queue plus mapped weight and KV memory, with the agent service holding the only capability. Needs the NPU vendor interfaces to be understood for the target SoC.
- Real-time guarantee: the agent service needs its cycle to start on time regardless of app load. Kalium's scheduler must provide a real-time class with priority over all app instances.
- Persistence: the window state (256 to 512 MB) must be checkpointed often enough that a crash loses little, and encrypted at rest.

## 10. System questions not about the model

- JS engine choice for the runtime shelf (Rust-native vs. C engine).
- Whether the IP stack lives in Kalium or in a service.
- Which permission grants are "sensitive" and need the user's own confirmation rather than the agent's.
- The widget description language and its renderer.
- Which open weights licence the model ships under.

## 11. Where these documents differ from the recorded decisions

Koseka's internal QualiaOS notes record decisions made on 2026-10-31 and 2026-11-02 that these documents were written without. Until a document is brought in line, the recorded decision wins. Remove an entry when the document it names is reconciled.

- **The model's shape** ([model-architecture.md](model-architecture.md)). Here: a chain of gated-attention layers at `d = 2048`, one layer per cycle, over a sliding StreamingLLM window. Recorded: Koseka's own model, a pool of routed single-layer blocks running one block per fixed 10 ms step and never reset. Still open on that side: one token per step instead of one per stream (proposed width about 1536, no wider than 2048), and whether weights stay in memory or rotate in from flash.
- **What the model perceives and does** ([agent-interface.md](agent-interface.md)). Here: a foveated view of the screen, live audio, articulatory speech, widgets and touch actions. Recorded: three vision streams (front camera, back camera, the website the model is looking at), two audio streams (website sound and the microphone) and its thinking stream; outputs are a retina per vision stream, one cursor, the keyboard and the articulatory voice, with no touch outputs. Still open: a widget output head, taps on widgets entering as input, and physical against neural rendering of the voice.
- **How the user is served** ([system-architecture.md](system-architecture.md), [web-apps-and-permissions.md](web-apps-and-permissions.md), the README). Here: every app is a web app, shown on screen. Recorded: the model reaches the user by voice, widgets and text; websites are driven by the model in the background and shown as they are only for occasional use by hand and for games; widgets come from templates, styled by packs from human designers rather than by the model.
- **What runs** ([system-architecture.md](system-architecture.md)). Here: WebAssembly only. Recorded: WebAssembly as the only runtime for apps, with natively compiled Rust next to it. JavaScript as an interpreter compiled to WebAssembly holds for apps; for websites it is open, to be measured against an engine with a JIT.
- **The kernel** ([system-architecture.md](system-architecture.md)). Here: Kalium as a kernel of Koseka's own that runs only WebAssembly, as its prototype does. Recorded: the kernel's base, a stripped-down Linux or a kernel of Koseka's own, is undecided, with the license following from it; see Kalium's `docs/status.md`.
