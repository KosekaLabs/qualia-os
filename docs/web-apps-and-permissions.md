# Web apps and permissions

In QualiaOS there is exactly one kind of application: a web app. This document
explains what that means in practice, how a web app gains the powers of a
native app, and how the runtime shelf keeps this cheap.

## A site is an app

Every app is identified by a URL (its origin plus an entry point). Visiting a
URL and "installing" an app are the same operation at different trust levels:

| Trust level | What the app can do |
| --- | --- |
| Visited | Render, run its code, use the network to its own origin and to origins it declares, store a bounded amount of data. Nothing else. Suspended when not on screen. |
| Pinned | Same as visited, plus it appears in the user's app list and its assets are cached for offline use. |
| Granted | Pinned, plus any permissions the user has explicitly approved. This is the level at which a web app behaves like a native app. |

There is no app store as a gatekeeper. Discovery and reputation are separate
concerns from execution and can be provided by services (including the
agent's own judgement) rather than by the OS.

## Manifest

An app declares what it is and what it would like in a manifest fetched from
its origin:

- Name, icon, theme hints for the glass UI.
- Entry assets: an HTML document, or a Wasm module for a Wasm-native app.
- Runtime requirements: which shelf modules (JS engine, HTML/CSS engine,
  others) and minimum versions it needs, or that it brings its own.
- Requested permissions, each with a human-readable reason.
- Background behaviour: whether it wants to run while not on screen and
  under what triggers (timer, push, network event, content handler).
- Agent interface (optional): a structured description of the actions the
  app exposes to the system agent, so the agent can operate the app directly
  rather than through the screen.

Requested permissions are not granted by declaring them. They are granted by
the user, at the time they matter, and they are revocable.

## Permissions become capabilities

Permissions are the user-facing vocabulary ("read my photos", "run in the
background", "use the microphone"). Capabilities are the kernel's vocabulary
(a handle that authorizes a set of host functions on a set of resources).
The permission manager is the only component that translates one into the
other:

```
 user approves "Photos: read"  ──► permission manager ──► mints capability
                                                           (file service, /photos, read)
                                                           and hands it to the app instance
```

Consequences:

- An app instance can only call host functions for capabilities it holds.
  Without a capability the import is simply not there; it is not a
  permission check that can be bypassed, it is an absence.
- Revoking a permission revokes the capability. Running instances lose the
  import on their next call.
- The agent can request permissions on the user's behalf, but sensitive
  grants require the user's spoken or touched confirmation. Which grants are
  "sensitive" is a policy decision documented in
  [open-questions.md](open-questions.md).

### Initial permission vocabulary

The list below is a starting point, not a final one.

| Permission | Enables |
| --- | --- |
| `background` | Keep running when not on screen; wake on declared triggers. |
| `files:<scope>` | Read and/or write a part of the user's file tree (per-folder, per-type, or everything). |
| `notifications` | Post notifications the agent may read out or show. |
| `microphone`, `camera`, `location`, `sensors` | Access the corresponding device through the relevant service. |
| `network:<pattern>` | Reach origins beyond the app's own and its declared ones. |
| `contacts`, `calendar`, ... | Structured user data held by system services. |
| `handler:<type>` | Register as the app that opens a content or link type. |
| `agent-interface` | Expose structured actions to the system agent and receive structured requests from it. |
| `system-ui` | Draw outside its own surface (status areas, lock screen widgets). Rarely granted. |

## The runtime shelf

Most web apps need a JavaScript engine and an HTML/CSS engine. If every app
shipped its own, each would pay in download size, memory and startup time,
and the kernel would still have to run them. Instead the OS ships these as
pre-compiled WebAssembly modules that sit next to the kernel.

- **Shared code, private state.** A shelf module's code is mapped read-only
  and shared by every app that uses it. Each app gets its own instance and
  its own linear memory. Compromising one app's JS engine instance gives
  nothing about another app's.
- **Versioned.** The shelf carries explicit versions. An app can require a
  minimum; the OS can carry more than one version during transitions.
- **Replaceable.** An app that needs an engine the shelf does not have, or a
  different one, ships it as an ordinary Wasm module. It works; it just
  does not get the sharing benefit.
- **Not kernel code.** The kernel does not special-case shelf modules. They
  are Wasm modules the app manager knows to load. Keeping this true is what
  keeps Kalium pure.

### Candidate shelf contents

Which engines to compile to Wasm is a practical question about which
projects compile well and what licensing we accept. Candidates:

| Role | Candidates |
| --- | --- |
| JavaScript | A Rust engine (Boa) for a clean Wasm build; a C engine (QuickJS, SpiderMonkey) for compatibility and performance. The choice is tracked in open questions. |
| HTML/CSS | Servo's component crates (html5ever, Stylo, Taffy for layout) assembled into a Wasm renderer; or a purpose-built lighter engine for the subset that mobile web apps actually use. |
| Other languages | CPython or RustPython, Lua, and any interpreter the community contributes. |

### How a page actually runs

1. The app manager fetches the manifest and entry assets.
2. It instantiates the HTML/CSS engine with the app's capabilities and a
   surface from the compositor.
3. The engine parses the document, and for each script asks the app manager
   for a JS engine instance bound to the same app and linear memory
   arrangement.
4. Script calls into the web platform (DOM, fetch, storage, device APIs)
   are implemented by the engines on top of host functions gated by the
   app's capabilities.
5. Painting goes to the surface; the compositor applies the glass styling
   and shows it.

To an app this looks like a browser. To the kernel it is a few Wasm
instances with some capabilities.

## Background execution

A granted `background` permission lets an app run while off screen, but
background is a budget, not a blank check. The app manager meters CPU, memory
and network for background instances and suspends them if they exceed the
budget, resuming them on their declared triggers. The agent is informed of
suspensions so it can explain them to the user if asked.

## Updates

Apps update the way websites do: by serving new assets. Pinned apps re-fetch
on a schedule or on push, verify integrity against the manifest, and swap at
a safe point (not while the user is in the app unless the app asks). The OS
itself, including the kernel and the runtime shelf, updates through signed
images handled outside the app model.
