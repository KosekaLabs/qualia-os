# The agent's senses and actions

This document describes the boundary between the model and the rest of the
OS: what goes into the encoder each cycle, and what the decoder's outputs
are allowed to do. The model internals are in
[model-architecture.md](model-architecture.md).

## Senses (encoder inputs)

Every inference cycle in which the encoder is called receives one vector made
by concatenating the following and projecting it to the model dimension.

### Foveated vision

The model does not see the full screen. A non-parametric foveation function
takes:

- a high-resolution capture of the screen, roughly phone-screen sized,
  rotated to portrait if the device is in landscape;
- a fixation point (x, y) and optionally a rotation, both chosen by the
  model on a previous cycle;

and returns a small image: high resolution near the fixation point, coarser
further away, with 1024 samples per color channel (3072 values). This is the
same trade the human eye makes: a cheap wide field plus an expensive detail
region that the model steers.

Two things follow from this. First, "where to look" is an action the model
takes, not something the OS decides. Second, the model must know where it is
currently looking, so the current fixation (and rotation) is also fed back as
part of the encoder input (see open questions).

The screen capture is of what the user sees: app surfaces after
composition, including the model's own widgets. The agent therefore sees its
own output, which is what makes it possible for it to verify that an action
had the intended effect.

### Live audio

The audio service delivers the microphone signal as fixed-length frames, one
per encoder call, as a vector of 1024 or 2048 samples (or a feature vector
of that size). The frame length and the sample rate together set the
model's inference cadence; see the budget in
[open-questions.md](open-questions.md). The audio frame includes what the
phone itself is playing, so the model hears its own voice and the apps'
audio, and can avoid talking over them.

### Its own last thought

The enriched embedding from the end of the previous cycle (2048 values). This
is what carries the model's state across encoder calls and is what makes the
loop closed rather than a stateless function of the senses.

### Optional structured inputs

Not in the baseline, but natural extensions of the same concatenation:
device state (battery, connectivity, time), the current fixation point,
notifications, and structured responses from apps that implement the agent
interface. Each extra input widens the encoder matrix by its dimension and
nothing else.

## Actions (decoder outputs)

The decoder reads the last enriched token and emits, every cycle:

1. **Routing**: which cognitive-chain layer runs next.
2. **Encoder call**: whether the next cycle should ingest fresh senses.
3. **Actions**: zero or more of the following.

### Speech via articulatory synthesis

The model does not generate audio samples. It emits a low-dimensional
trajectory of articulator parameters (vocal tract shape, glottal source,
pitch, intensity) at the inference cadence, and a fixed articulatory
synthesizer in the audio service turns that trajectory into sound.

Why this instead of a neural vocoder or raw audio:

- The output is tiny (tens of parameters per frame), so it fits the
  "one cycle, one small output" shape of the model.
- It is physically constrained. The synthesizer cannot produce sounds a
  vocal tract cannot, which keeps the voice stable and expressive without
  the model having to learn acoustics.
- It is naturally streaming and real-time, matching the inference cadence.
- The voice is a property of the synthesizer's vocal-tract model, so the
  user can change the voice without changing the model.

### Widgets

When audio is the wrong channel (a list to pick from, a map, a chart, a
confirmation with details, anything the user needs to look at rather than
hear), the model emits a declarative widget description and the compositor
renders it in the glass design system.

The approach follows A2UI: the model produces structure and content, never
pixels or markup, and the renderer owns appearance. We extend it in two
ways:

- **Range.** The widget vocabulary is large enough, and composable enough,
  that the model can produce almost any interface a mobile app would show:
  lists, cards, forms, media, maps, charts, tables, timelines, progress,
  and layouts of those. The goal is for the user to feel the model can
  draw whatever is needed.
- **Design.** Every widget is styled by the system, so all generated UI is
  coherent with each other and with the user's chosen look, with blur,
  gradients and motion handled by the compositor.

Widgets are interactive. A tap on a widget comes back to the model as a
structured event through the same path as other senses. Widgets are also
ephemeral by default: they exist for the task that created them and are
dismissed when it ends, unless the user pins them.

Widgets are the exception, not the rule. The model is trained to prefer
speaking, and to generate a widget only when it would convey the
information better than audio.

### Phone actions

The model operates apps the way a user would:

- Touch: tap, long press, drag, swipe, pinch at screen coordinates.
- Type: text input into the focused field.
- Navigate: open a URL or app, go back, switch app, go home.
- System: adjust volume and brightness, toggle radios, read and dismiss
  notifications, start and stop media.
- Permissions: request a permission on behalf of the user (sensitive ones
  need the user's confirmation).
- Agent interface calls: structured requests to apps that expose one.

Phone actions go through the input service and app manager, which apply the
same checks they would for a human user.

### Where to look

The fixation point and rotation for the next foveated capture. This is
emitted every cycle in which the encoder will be called, and is the model's
way of "scrolling its eyes" over a page.

## Latency and turn-taking

Because the model runs continuously at a fixed cadence and hears itself,
there is no explicit turn-taking protocol. The model decides when to speak
by emitting non-silent articulator parameters, and when to stop by going
silent, exactly as a person does. Interruptions work the same way: the model
hears the user start talking and can choose to stop.

## Privacy boundaries

The agent sees the screen and hears the microphone at all times, which is
what makes it useful and what makes it sensitive. The constraints we commit
to:

- Everything the model perceives stays on the device. Inference is local.
- The model's context state is stored encrypted at rest with the device key.
- The user can mute senses (screen, mic) and the model is told it has been
  muted rather than being fed silence.
- Apps cannot query the agent's state or perceptions; the agent interface is
  one-directional from the agent's point of view.
