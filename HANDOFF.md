# Handoff

This repository holds the QualiaOS design documents (see `docs/`). The implementation work so far is the Kalium kernel prototype; its full handoff (state, how to run it, open issues, next steps) is `HANDOFF.md` in the [Kalium repository](https://github.com/KosekaLabs/kalium).

## Where the work stands (2026-11-10)

- The first session ran in the cloud and delivered its work as two git bundles. Both were unpacked and verified against their final commits, then removed; the work now lives only in git.
- Kalium: 16 commits, local checkout on `dev`.
- QualiaOS: 4 commits of design documents on top of the initial commit, merged into `dev`.
- Nothing has been pushed yet. Both repositories merge into `dev`, never `main` (Phased Versioning); `v1` will be cut from `dev` later.

## Open items

- The model described in `docs/model-architecture.md` and `docs/open-questions.md` differs from the latest decisions recorded in Koseka's internal notes; the owner decides which one wins before these documents are treated as the reference.
- This repository has no Rust code yet, so it has no `Cargo.toml` and no CI workflow. Both are added when code lands.
