# QualiaOS: instructions for agents

These are the standing instructions for any agent (or person) working in this repository. They always apply. Where the work stands is in the README's status section, and the decisions still open, including how these documents must be reconciled with the decisions already recorded, are in [docs/open-questions.md](docs/open-questions.md): read both before you start.

## What QualiaOS is

QualiaOS is Koseka's open-source mobile operating system, and the AI model inside it: the OS is the model. It is installed in place of a phone's own OS, written in Rust, voice-first, and runs its model continuously on the phone. The model's weights are published with the OS; how the model is trained is not (see "Never" below).

Its kernel is [Kalium](https://github.com/KosekaLabs/kalium), which lives in its own repository with its own instructions and its own status page (`docs/status.md` there), including the open decision on the kernel's base.

## What this repository holds

Design documents only, for now: `docs/` (system architecture, web apps and permissions, model architecture, the agent's interface, open questions), plus the README and the community files. There is no code, so there is no `Cargo.toml` and no CI workflow yet. When the first Rust code lands, add `Cargo.toml` from Koseka's manifest template and the standard CI/CD workflow (as Kalium has), and update this file.

Koseka's internal QualiaOS notes are the record of the product's decisions. The documents here were written without some of those decisions and differ from them in places, listed at the end of `docs/open-questions.md`. Until a document is reconciled, the recorded decision wins; when you reconcile one, remove its entry from that list in the same change.

## Build, lint and check

- `npm install` once, then `npm run trunk git-hooks sync` to install the hooks.
- `npm run check` runs the trunk metalinter; `npm run fmt` formats. The pre-commit hook formats and the pre-push hook checks.
- Prettier formats Markdown tables. A `|` inside a code span in a table must be escaped as `\|`, or the table breaks.

## Repository conventions

- **Branches and versions:** [Phased Versioning](https://labs.koseka.net/phased-versioning), never SemVer. `dev` is the default and nightly branch, the only place breaking changes may land; there is no `main` or `master`. Version branches (`v1`, `v2`, …) are cut from `dev` and receive `dev` merges only. Until `v1` exists, the project is in its zeroth generation (version `0.0.0`).
- **Releases** go through the [`oyo`](https://github.com/AmonRayfa/oyo) CLI (`oyo gen`, `oyo phase <name>`, `oyo rev`), never raw `git branch` or `git tag` for version branches and release tags. A release adds its CHANGELOG entry on `dev`, merges `dev` into the version branch, then tags with `oyo`.
- **Commits** follow [Conventional Commits](https://www.conventionalcommits.org) and explain why.
- **Rust**, once there is code: edition 2024, formatting with `.trunk/configs/.rustfmt.toml`, and error handling with [Mabe](https://github.com/AmonRayfa/mabe) (`#[mabe::main]` on the entry point, its re-exported `bail`, `Context` and `Result`, and its `Error` derive for custom error enums).
- **License:** Apache 2.0, stock text (`LICENSE`), with `NOTICE` and the community files (README, CONTRIBUTING, CHANGELOG, SECURITY) kept current.
- **Names:** Kalium is the kernel (it also hosts the peer-to-peer layer). Kora is a separate desktop companion app, not part of QualiaOS. Do not use either name for anything else.

## Product rules

- The agent runs only on the phone. Other devices are displays and inputs driven by the phone, never hosts of the agent.
- A user's memory and personal data stay on their device.
- Face and voice recognition are only ever on-device, opt-in and consented, with no scraping and no shared biometric databases.
- The model is described as "open weights", never "open source AI", because its training code is not published.

## Never

- Never describe, hint at or commit how the model is trained: no method, training code, training results or training framework details, in documents, code, comments, commit messages or issues. Saying nothing is the rule; never give a false account either.
- Never commit code or files you do not have the right to publish under Apache 2.0, including code from employers or other private projects. Re-implement ideas instead of copying files. This repository is public.
- Never commit secrets or credentials.
- Never create a `main` branch, commit straight to the newest version branch, or rewrite `dev`'s published history.
- Never tag or cut versions by hand; use `oyo`.
