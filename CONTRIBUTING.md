# Contributing

This repository is for focused technical discussion of a possible Rust async-drop endgame design. It is a design note, not an RFC, so contributions should improve clarity, identify risks, or make the tradeoffs easier to evaluate.

## Helpful contributions

- Mistakes in the current documents.
- Missing prior art.
- Unsoundness risks.
- Improved examples.
- Wording clarity.
- Alternative migration paths.

## Discussion style

Keep feedback technical and specific. When possible, point to a section, quote the relevant claim briefly, and explain the concrete concern or suggested change.

Good issues usually include one of:

- a minimal example
- a counterexample
- a precise wording improvement
- a comparison to existing Rust behavior
- a specific prior-art reference

Broad design conversation is welcome, but it usually belongs in GitHub Discussions or Zulip rather than in a narrow issue.

## Scope

In scope:

- async destruction semantics
- synchronously undroppable types
- cancellation behavior for implicit async cleanup
- `Drop` and `AsyncDrop` interaction
- dynamic dispatch and allocation concerns
- migration and diagnostics
- wording and structure of the design note

Out of scope:

- executor-specific APIs unless they affect implicit async destruction
- general cancellation design outside async drop
- unrelated async Rust ergonomics
- unstable syntax bikeshedding without a semantic issue

## Licensing

This repository is dual licensed under MIT OR Apache-2.0.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this repository is licensed as MIT OR Apache-2.0, without additional terms or conditions.
