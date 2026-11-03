# QualiaOS system architecture

This document describes the layers of QualiaOS and the responsibilities of
each. It is the top-level map; the other documents in this folder zoom into
individual parts.

## Design principles

- **One execution format.** Everything that executes above the hardware is
  WebAssembly. The kernel never loads native code from the user space, and
  there is no second runtime for "real" apps.
- **Keep the kernel pure.** Kalium knows about memory, scheduling,
  capabilities, devices and WebAssembly. It does not know what JavaScript,
  HTML or an "app" is.
- **Web apps are the only apps.** A site and an installed app are the same
  thing; the only difference is which permissions the user has granted.
- **Agent first, screen second.** The model is a system service with
  real-time scheduling, not an app. The user's primary interface is voice;
  the screen is for what voice cannot do well.
- **Pre-compile what everyone needs.** Interpreters and engines that most
  web apps need (JavaScript, HTML, CSS) are compiled to WebAssembly once,
  shipped with the OS and shared, so apps do not download or keep their own
  copies.

## Layer diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│  Web apps (sites with or without granted permissions)               │
│  HTML/CSS/JS assets · Wasm-native apps · any language → Wasm        │
├─────────────────────────────────────────────────────────────────────┤
│  Runtime shelf (shared, pre-compiled Wasm modules shipped with OS)  │
│  JS engine · HTML/CSS layout & paint · other interpreters           │
├─────────────────────────────────────────────────────────────────────┤
│  QualiaOS system services (privileged Wasm modules)                 │
│  Compositor/UI · Audio · Permissions · Files · Network · App mgr    │
│  Agent service (sensing → model → actions) · Widget renderer        │
├─────────────────────────────────────────────────────────────────────┤
│  Kalium kernel (Rust, bare metal, runs only WebAssembly)            │
│  Boot · MMU · Scheduler (RT class) · Capabilities · Wasm runtime    │
│  Device drivers incl. NPU, display, audio, touch, radios            │
├─────────────────────────────────────────────────────────────────────┤
│  Hardware: mobile SoC (CPU, GPU, NPU), display, mic/speaker, radios │
└─────────────────────────────────────────────────────────────────────┘
```

## Layer 0: hardware

Target class: a modern phone SoC with a high-end NPU. The model is designed
to run only on the NPU, so the NPU's throughput and, more importantly, its
memory bandwidth bound what the model can be (see
[open-questions.md](open-questions.md)). The CPU runs the kernel and all
WebAssembly; the GPU is used by the compositor for the glass UI (blur,
gradients) and is available to the HTML/CSS engine for painting.

## Layer 1: Kalium kernel

Kalium is documented in its own repository. From QualiaOS's point of view it
provides:

- **Wasm instances as the unit of isolation.** What other systems call a
  process is a WebAssembly instance (or component) with its own linear
  memory. Isolation comes from the Wasm sandbox and from the MMU together.
- **Capabilities.** Every host function an instance can call is gated by a
  capability the instance holds. Capabilities are unforgeable handles granted
  at instantiation or passed over IPC. Permissions in the user-facing sense
  are built on top of this.
- **Scheduling with a real-time class.** The agent service runs on a fixed
  inference cadence and must not be starved by apps. The kernel offers a
  real-time scheduling class for it and for the audio pipeline.
- **Device drivers**, including the NPU driver that exposes a command queue
  and memory mapping for the model's weights and context state.
- **IPC** between instances, used by services to talk to each other and to
  apps.

## Layer 2: system services

System services are WebAssembly modules like everything else, but they are
instantiated at boot with capabilities ordinary apps never get (raw display,
raw audio, raw input, NPU access, the whole file system). They are the parts
of a conventional OS that live outside the kernel.

| Service | Role |
| --- | --- |
| Compositor | Owns the display. Composes app surfaces and system widgets with the glass design system (blur, color gradients, user-customizable). |
| Audio | Owns mic and speaker. Delivers live audio frames to the agent service, plays the articulatory synthesizer's output, mixes app audio. |
| Input | Owns touch, buttons and sensors. Routes events to apps or to the agent. |
| Permission manager | Maps user-level permissions ("this app may read my photos") to kernel capabilities. The single place where capabilities are minted for apps. |
| File service | Virtual file system over storage. Apps see what their permissions allow. |
| Network | TCP/IP, TLS, HTTP fetch, DNS. Whether the IP stack lives here or in Kalium is an open question. |
| App manager | Resolves URLs to apps, fetches and caches assets, instantiates runtime-shelf modules for them, handles lifecycle (foreground, background, suspended). |
| Agent service | Feeds the model (screen foveation, audio, state), runs the inference loop on the NPU, executes the model's actions, renders its widgets. See [agent-interface.md](agent-interface.md). |

## Layer 3: runtime shelf

The runtime shelf is the set of pre-compiled WebAssembly modules that the OS
ships for interpreting web content and other languages. It is the central
trick that lets the kernel stay pure while the OS still runs the web.

- **JavaScript engine** compiled to Wasm. The JS files of a site are assets
  the engine loads; the kernel never sees JavaScript.
- **HTML/CSS engine** compiled to Wasm: parsing, style, layout, painting
  into a surface the compositor shows.
- **Other interpreters** (Python, Lua, ...) can be added to the shelf so
  apps written in those languages do not have to ship an interpreter.

Shelf modules are loaded once and shared read-only; each app gets its own
instance with its own linear memory, so apps stay isolated while the code is
shared. Apps may still bring their own interpreter or engine as an ordinary
Wasm module if they want a version the shelf does not offer. Details are in
[web-apps-and-permissions.md](web-apps-and-permissions.md).

## Layer 4: web apps

A web app is a URL. Loading it fetches a manifest and assets, the app manager
picks the runtime-shelf modules it needs (or none, for a Wasm-native app),
and the permission manager grants whatever the user has approved. "Installing"
an app means pinning it and granting it permissions that let it behave like
a native app: run in the background, access files, receive notifications,
register as a handler for a content type, and so on.

## How the agent fits in

The agent is not an app and not the shell. It is a system service that sits
alongside the compositor and audio services and sees the same things the user
sees:

```
 mic ──► audio frames ──┐
                        ├─► encoder ─► cognitive chain ─► decoder ─┬─► speech (articulatory synth)
 screen ─► foveation ───┘                     ▲                    ├─► widgets (declarative UI)
                        last enriched token ──┘                    ├─► phone actions (touch, type, navigate)
                                                                   └─► where to look next (fovea)
```

The agent acts on apps the same way a user would: through the input service
(touch, typing) and through the app manager (open URL, switch app). It does
not get a privileged backdoor into app internals. This keeps apps honest and
means anything the agent can do, the user could have done by hand. Where an
app wants to be a better citizen for the agent, it can expose a structured
interface through a permission, but that is an optimization, not the
baseline.

The agent is also the primary UI. By default the screen shows the ambient
glass surface and whatever widgets the model has produced; apps are brought
to the foreground when the user or the agent asks for them.

## Boot sequence (intended)

1. Firmware hands control to Kalium.
2. Kalium brings up memory, the scheduler, core drivers and the Wasm runtime.
3. Kalium instantiates the system services from the boot image with their
   capabilities, starting with permission manager, compositor, audio and
   input.
4. The agent service maps the model weights into the NPU, restores the
   persisted context state (the model is never reset; see
   [model-architecture.md](model-architecture.md)) and starts the inference
   cadence.
5. The app manager restores pinned apps that have background permission.
6. The phone is ready; the user talks to it.

## What is deliberately not here

- No native app binaries, no dynamic linking of native libraries, no shell
  with a conventional process model.
- No extension or plugin system. Capabilities granted to a web app cover
  what extensions do elsewhere.
- No separate "assistant app". The agent is part of the OS.
