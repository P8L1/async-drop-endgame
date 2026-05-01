# Async Drop Endgame

This repository contains a design note exploring a possible end-state for Rust async drop. It is not an RFC, and it is not intended to claim consensus. The goal is to make one coherent model concrete enough for technical feedback.

The proposal builds on existing Rust async-drop work and focuses on:

- synchronously undroppable types and a possible `!Destruct` capability boundary
- cancellation behavior for implicit async-drop await points
- generated futures that may themselves need async destruction
- interaction between `Drop` and `AsyncDrop`
- async destruction of `dyn Trait` values without mandatory extra allocation
- a staged migration path from today's explicit cleanup patterns

Start here:

- [Main design note](async-drop-endgame.md)
- [Open questions](open-questions.md)
- [Prior art](prior-art.md)
- [Contributing](CONTRIBUTING.md)

## Scope

This repo is for discussing the shape of a Rust language design in which async cleanup is treated as a first-class part of destruction. It tries to describe compiler behavior, type-system boundaries, dynamic dispatch concerns, and migration steps at a level useful for community review on GitHub and Zulip.

## Design goals

- Make required async cleanup expressible in the type system.
- Preserve predictable Rust destruction order.
- Avoid mandatory hidden heap allocation for ordinary static async destruction.
- Keep costs visible where possible.
- Fit with `Pin`, `Send`, auto traits, and existing async lowering.
- Leave room for incremental stabilization.

## Non-goals

- This is not a complete RFC.
- This does not specify exact syntax for every feature.
- This does not try to settle executor APIs or cancellation models outside implicit async destruction.
- This does not replace existing Rust async-drop tracking, roadmap, or design work.

## Feedback

Specific technical feedback is welcome. Useful feedback includes unsoundness concerns, examples that break the model, missing prior art, unclear wording, and alternative migration paths.

Use GitHub issues for focused corrections or design feedback. Use GitHub Discussions or Zulip for broader exploratory conversation.

## License

This repository is dual licensed under either Apache-2.0 or MIT, at your option.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this repository is licensed as MIT OR Apache-2.0, without additional terms or conditions.
