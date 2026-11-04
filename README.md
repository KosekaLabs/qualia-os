# QualiaOS

QualiaOS is an open source, agentic mobile operating system. It is built on
[Kalium](https://github.com/koseka/kalium), a Rust kernel that runs only
WebAssembly, and it ships with a built-in, open-weights AI model that runs on
the phone's NPU and acts on the user's behalf.

## The idea in three sentences

1. **The browser replaces the operating system.** Every app is a web app, with
   or without extra permissions granted by the user. There are no native apps,
   plugins or extensions.
2. **The kernel only runs WebAssembly.** JavaScript, HTML and CSS are not
   special: their interpreters and engines are themselves WebAssembly modules
   that ship with the OS, and web content is just assets those modules load.
   Any language that compiles to WebAssembly, or whose interpreter does, is a
   first-class citizen.
3. **Humans stop navigating the web; the agent does it for them.** The
   interface is audio-centric. The user talks to the phone, the model talks
   back through articulatory speech synthesis, and it generates on-screen
   widgets only when audio is the worse way to convey something.

## Documentation

| Document | What it covers |
| --- | --- |
| [docs/system-architecture.md](docs/system-architecture.md) | The layers of the OS, from Kalium to web apps, and how the agent fits in. |
| [docs/web-apps-and-permissions.md](docs/web-apps-and-permissions.md) | How web apps become "apps", the capability and permission model, and the shared runtime shelf of pre-compiled interpreters. |
| [docs/model-architecture.md](docs/model-architecture.md) | The built-in model: encoder, cognitive chain, decoder, the inference cycle and the sliding context window. |
| [docs/agent-interface.md](docs/agent-interface.md) | What the model perceives and what it can do: foveated vision, live audio, articulatory speech, widgets and phone actions. |
| [docs/open-questions.md](docs/open-questions.md) | Dimension checks, budgets, and the design decisions that still need to be made. |

The kernel has its own documentation in the
[Kalium repository](https://github.com/koseka/kalium).

## Status

Design phase for QualiaOS itself; the documents above are the reference for
what we intend to build and are expected to change as decisions in
[docs/open-questions.md](docs/open-questions.md) get resolved.

The kernel is further along: the first [Kalium](https://github.com/koseka/kalium)
prototype boots on QEMU (aarch64), runs WebAssembly services through a
spec-tested interpreter, and demonstrates capability-gated host calls, IPC
with capability transfer, and real-time plus fair scheduling. See Kalium's
`docs/running.md` for how to run it.

## License

Apache License 2.0. See [LICENSE](LICENSE).
