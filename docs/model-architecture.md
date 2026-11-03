# Model architecture

QualiaOS ships with a built-in, open-weights model that runs on the phone's
NPU. This document describes its architecture as currently intended. The
training method is closed source and is not covered here.

The dimensions and budgets below are the baseline; the places where they
still need to be tuned are collected in
[open-questions.md](open-questions.md).

## A Turing machine, not a chatbot

The model is best understood as a Turing machine. It has a tape, a head, and
a small set of states, and all it ever does is read and write:

| Turing machine | This model |
| --- | --- |
| Tape | The sliding context window of token embeddings (plus, eventually, external memory). |
| Head position | The last token in the window; the model always works at the end of the tape. |
| Read | Attention over the window. |
| Write | Appending a new enriched embedding, and emitting actions. |
| State | The last enriched embedding. |
| Transition function | The decoder layer, which picks the next cognitive-chain layer to run. |

Unlike a conventional language model there is no fixed forward pass. The
model runs one layer per inference cycle, forever, and decides for itself
which layer comes next and when to look at the world again.

## Components

```
                       ┌────────────────────────────────────────────┐
  senses ──────────────► encoder (linear)                           │
  (foveated image,      │   ▲                                       │
   audio frame,         │   │ called only when the decoder says so  │
   last embedding)      └───┼───────────────────────────────────────┘
                            │                   
                     ┌──────▼──────────────────────────────┐
                     │ cognitive chain (CC)                │
                     │ N gated-attention layers L_1 … L_N  │
                     │ one layer runs per inference cycle  │
                     └──────┬──────────────────────────────┘
                            │ last enriched token
                     ┌──────▼──────────────────────────────┐
                     │ decoder (router + output head)      │
                     │ → next CC layer                     │
                     │ → call encoder?                     │
                     │ → actions (speech, widgets, touch…) │
                     └─────────────────────────────────────┘
```

### Encoder

The encoder is a single linear layer, preceded by the non-parametric
foveation function. Its input is the concatenation of:

| Input | Dimension |
| --- | --- |
| Foveated screen image, 1024 samples per color channel | 3072 |
| Live audio frame | 1024 or 2048 |
| Last enriched embedding from the previous cycle | 2048 |
| **Total** (with 2048 audio) | **7168** |

It projects this to the model dimension (2048), giving a 7168 × 2048 matrix
(about 14.7M parameters). The output is appended to the context window as a
new token. The foveation function itself has no parameters; it is a fixed
resampling of the screen around a fixation point the model chose.

The encoder is called only when the decoder decides it should be. Between
encoder calls the model keeps refining the last token without new sensory
input. This is how the model "thinks" for several cycles about one
observation.

### Cognitive chain

The cognitive chain (CC) is a set of N gated-attention layers. Each layer
follows the same equation:

```
X'_i = ( SiLU(X_i W_g) ⊙ concat_h( softmax( X_i W_q^h (X_j W_k^h)^T ) X_j W_v^h ) ) W_o
```

where ⊙ is the Hadamard product, `h` ranges over the heads, `i` is the
token being enriched and `j` ranges over the tokens it may attend to
(causal: `j ≤ i`). Note that keys and values both come from the context
positions `j`; the value projection applies to `X_j`, not `X_i`.

| Symbol | Shape | Role |
| --- | --- | --- |
| `d` | 2048 | Model dimension. |
| `H` | 8 | Attention heads per layer. |
| `d_k = d_q = d_v` | 512 | Per-head key, query and value dimension. `H × 512 = 4096 = 2d`. |
| `W_g` | 2048 × 4096 | Gate projection. Projects up by a factor of 2 so the gate matches the concatenated heads. |
| `W_q, W_k, W_v` | 2048 × 4096 each | Per-head projections stored as one matrix each. |
| `W_o` | 4096 × 2048 | Output projection back to `d`. |

Per layer this is 5 matrices of 2048 × 4096, about 41.9M parameters. There
is no separate feed-forward block; the SiLU gate on the attention output is
the layer's nonlinearity. The layers have no fixed order: the decoder picks
which one runs next, so a layer can run twice in a row, be skipped, or be
visited in any sequence the model has learned.

Attention is strictly causal self-attention over the context window.

### Decoder

The decoder is applied to the last enriched token after every CC layer. It
is three things at once:

1. **Router.** A distribution over the N CC layers; the argmax (or a sampled
   choice) is the layer that runs in the next cycle.
2. **Encoder gate.** A binary decision: should the next cycle begin by
   calling the encoder and appending a fresh sensory token?
3. **Output head.** The model's actions: articulator parameters for speech,
   widget descriptions, touch and navigation commands, and the next fixation
   point. The output vocabulary is described in
   [agent-interface.md](agent-interface.md).

The decoder is also the model's first layer. At birth the context holds a
single random embedding. The decoder is applied to it, picks the first CC
layer, decides whether to call the encoder, and the loop begins.

## The inference cycle

One inference cycle is: run one CC layer on the last token, then run the
decoder. It takes a fixed amount of wall-clock time, set so that the cadence
matches the audio frame rate (see
[open-questions.md](open-questions.md) for the actual numbers).

```
loop forever, once per cycle:
    if decoder said "call encoder":
        s   ← [ foveate(screen, fixation), audio_frame, x_last ]
        x   ← s W_enc
        append x to the window           # write
    layer ← decoder's choice
    x_last ← layer(x_last, window)       # read (attend) + refine
    route, call_encoder, actions ← decoder(x_last)
    execute actions
```

Within a cycle, attention reads over the whole window but only the last
token is updated. Older tokens are never rewritten; the model's only way to
change what it will remember is to write new tokens.

## The window never resets

The model is a real-time system. Its context window slides forever
following the StreamingLLM mechanism: a small set of attention-sink tokens
at the start of the window is kept permanently, and behind them a rolling
window of the most recent 32k tokens. When a new token is appended the
oldest non-sink token is dropped. Position information is relative to the
cache, not to absolute time, so the model never sees a position it was not
trained on.

Because the model is never reset, its state (the window contents and the
last embedding) is persisted across reboots and restored on boot. There is
no "new conversation". The model's life is one continuous stream, and the
window is its short-term memory. What it keeps for longer than the window
is one of the open questions.

## Why this shape

- **One layer per cycle** gives a fixed, small per-cycle cost, which is what
  makes a hard real-time cadence possible on an NPU.
- **Learned routing** replaces a fixed depth with adaptive depth: easy
  moments cost one or two cycles, hard ones can spend hundreds, and the
  model chooses.
- **Encoder on demand** means the model samples the world when it needs to
  rather than being flooded with a sensory token every cycle, and it keeps
  the window from filling with redundant observations.
- **Gated attention without an FFN** keeps every parameter in the part of
  the layer that reads the tape, which fits the Turing-machine framing and
  keeps the per-layer size and shape uniform, which the router needs.
- **Articulatory output** keeps the output per cycle tiny and continuous,
  so speaking is just another thing the decoder does every cycle rather
  than a separate generation loop.
