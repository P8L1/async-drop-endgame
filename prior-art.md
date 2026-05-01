
# Prior Art

This repository builds on existing Rust async-drop directions. The links below are the intentionally limited reference set for this note.

## Rust async-drop tracking issue

https://github.com/rust-lang/rust/issues/126482

The tracking issue is the central place to follow Rust compiler and language work related to async drop. It provides context on implementation status, feature-gate discussion, and the kinds of unresolved language questions that remain.

## Rust async-drop roadmap

https://rust-lang.github.io/async-fundamentals-initiative/roadmap/async_drop.html

The async fundamentals roadmap explains why async destruction matters for the broader async Rust effort. It helps frame async drop as part of making async resource management reliable rather than as an isolated feature.

## Zetanumbers async-drop design

https://zetanumbers.github.io/book/async-drop-design.html

This design write-up explores async drop in more detail, including destructibility, generated destructor futures, and the relationship between async cleanup and Rust's existing destruction model. This repo draws from that direction while keeping open which exact syntax and staging would be best.

## Pin ergonomics goal

https://rust-lang.github.io/rust-project-goals/2025h2/pin-ergonomics.html

Pin ergonomics are relevant because async destruction likely needs to operate on pinned values. Improvements in this area may affect how `AsyncDrop` can be implemented, projected, and taught.

## Relationship to this repository

This repo does not try to supersede those efforts. It proposes one possible end-state that combines first-class async destruction, synchronously undroppable types, cancellation-shielded implicit cleanup, and allocation-conscious dynamic dispatch into a single discussion target.
