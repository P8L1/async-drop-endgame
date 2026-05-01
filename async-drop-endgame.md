
# Async Drop Endgame: First-Class Destructibility and Cancellation-Shielded Cleanup

## 1. Status

This is a design note, not an RFC. It is intended to explore one possible end-state for Rust async drop in enough detail to make tradeoffs discussable.

The note builds on existing Rust async-drop work. It does not try to replace that work or claim that the design here is settled.

## 2. Summary

Rust should treat async destruction as a compiler-known sibling of normal destruction.

The core pieces are:

1. compiler-generated async drop glue for all types
2. a user-facing `AsyncDrop` trait for custom async cleanup
3. type-system support for synchronously undroppable types
4. cancellation-shielded implicit async-drop await points
5. dynamic dispatch support without mandatory extra heap allocation

The intended model is direct: synchronous code requires synchronous destruction, while async code can perform async destruction. Some types may eventually be expressible as async-only destructible.

## 3. Motivation

Today, async cleanup is often represented by explicit `.close().await`, `.flush().await`, `.shutdown().await`, or `.rollback().await` methods. That pattern is workable, but fragile. It depends on every caller remembering to call the method on every control-flow path, including early returns and error paths.

Examples include:

- async I/O flush before a writer is discarded
- network shutdown that sends protocol-level close frames
- distributed lock release that must contact a coordinator
- transaction rollback or commit
- protocol close handshakes where dropping a handle silently is not equivalent to closing it

Rust already makes synchronous cleanup reliable by tying it to ownership and scope exit. Async cleanup needs a similarly principled place in the language if some resources cannot be safely cleaned up by synchronous `Drop` alone.

## 4. Design goals

- Soundness: required cleanup should not depend only on convention.
- Explicit costs: async destruction introduces await points and should be visible in type and control-flow reasoning.
- No mandatory hidden heap allocation: compiler-generated async destruction for statically known types should not require boxing.
- Predictable destruction order: async destruction should preserve ordinary Rust destruction-order rules.
- Compatibility with `Pin`: pinned values need a destruction path that does not move them.
- Clear interaction with `Send` and auto traits: async destructor futures affect the futures that may run them.
- Incremental migration: synchronously droppable types should be able to adopt async cleanup before async-only destruction is introduced.
- Useful diagnostics: users should get direct explanations when a value cannot be dropped in the current context.

## 5. Core model

Normal Rust has synchronous drop glue. For a type `T`, the compiler knows how to destroy `T`: run user `Drop` if present, then destroy fields in the correct order.

Async Rust should have async drop glue. For a type `T`, the compiler should be able to construct an async destruction operation: run user async cleanup if present, otherwise run synchronous cleanup when applicable, then recursively async-destroy fields.

Conceptually, the compiler might reason in terms of an internal trait like:

```rust
trait AsyncDestruct {
    type AsyncDestructor: Future<Output = ()>;
}
```

This sketch is not proposed surface syntax. It is a way to describe the compiler-known relationship between a type and its async destructor future.

## 6. Async destruction algorithm

In an async context, destroying a value of type `T` should behave conceptually as follows:

1. If `T` implements `AsyncDrop`, run it.
2. Else if `T` implements `Drop`, run normal `Drop`.
3. Then recursively async-destroy fields.
4. Preserve ordinary Rust destruction-order rules.

This keeps async destruction recognizable as Rust destruction. The async version extends the cleanup operation with awaitable steps; it should not invent a surprising new ordering model.

## 7. Destructibility as a type-system property

The type system may need to distinguish among several properties:

- can be dropped synchronously
- can be dropped asynchronously
- cannot be safely dropped synchronously

A possible spelling could look like this:

```rust
trait Destruct {}
impl !Destruct for Connection {}
```

The exact syntax is open. The important property is semantic: if a type requires async cleanup for sound or specified behavior, accidentally allowing it to fall back to synchronous destruction should be a type-system error, not merely a best-effort warning.

## 8. Context rules

Sync contexts require sync destructibility. If a function may need to destroy a value without awaiting, the value must be synchronously destructible.

Async contexts can own values that are not synchronously destructible, because scope exit can run async destruction.

For example:

```rust
fn sync_consume<T: Destruct>(value: T) {}

async fn async_consume<T: ?Destruct>(value: T) {}
```

This syntax is only a sketch. The core rule is that async-only destructible values must not escape into contexts where the compiler cannot run their async destructor.

## 9. Why lints are not enough

A lint is useful for migration. It can identify types that probably should be explicitly closed, warn when an async cleanup method is ignored, or prepare users for a future stronger model.

As a final model, a lint is not enough. If async cleanup is required, sync drop should be a type error. Otherwise the language still permits the very bug the design is meant to prevent.

## 10. Cancellation behavior

Implicit async drop creates hidden await points. If those await points are cancellable in the ordinary way, cleanup can be interrupted after the program has already committed to destroying the value. That would make implicit cleanup much less reliable than synchronous drop.

The default should be that implicit async-drop await points are cancellation-shielded. Once destruction starts, the compiler-generated cleanup path should be driven to completion unless the process aborts or an equivalent unrecoverable condition occurs.

Explicit async cleanup remains normally cancellable:

```rust
resource.close().await;
```

Calling an explicit method is ordinary async control flow. Implicit destruction is different because it is part of ownership cleanup.

## 11. `Drop` and `AsyncDrop`

There are at least three useful strategies.

Synchronous only:

```rust
impl Drop for TempFile {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.path);
    }
}
```

Synchronous fallback plus better async cleanup:

```rust
impl Drop for BufferedWriter {
    fn drop(&mut self) {
        self.mark_unflushed_for_recovery();
    }
}

impl AsyncDrop for BufferedWriter {
    async fn drop(self: Pin<&mut Self>) {
        self.flush().await;
    }
}
```

Async-only cleanup:

```rust
impl !Destruct for Connection {}

impl AsyncDrop for Connection {
    async fn drop(self: Pin<&mut Self>) {
        self.shutdown().await;
    }
}
```

The second strategy is important for migration. Existing types can keep a conservative synchronous fallback while offering better async cleanup in async contexts.

## 12. Dynamic dispatch

`dyn Trait` is hard because the concrete destructor future type is erased. For statically known `T`, the compiler can produce a concrete async destructor future. For a trait object, the destructor future depends on the erased concrete type.

Mandatory boxing should not be the default design target. Boxing every dynamic async destructor would make dynamic dispatch work, but it would impose allocation even when a chosen dynamic representation could avoid it.

The target property should be:

- static async destruction has no allocation
- dynamic async destruction has no mandatory extra allocation beyond the chosen dynamic representation

This may require vtable support, in-place erased future storage, placement-style protocols, or other compiler/runtime representation work. The key point is that allocation should be a representation choice, not the required semantic model.

## 13. Pinning

`AsyncDrop` likely needs a pinned receiver:

```rust
trait AsyncDrop {
    async fn drop(self: Pin<&mut Self>);
}
```

That shape allows async cleanup for pinned resources without moving the value. It also aligns with the fact that async destructor futures may hold references into the value while cleanup is in progress.

## 14. Auto traits and `Send`

If an async function owns `T`, the returned future may need to async-drop `T`. That means the outer future's `Send` status can depend on the async destructor future for `T`.

This follows the same broad principle as ordinary async lowering: values held across await points affect the generated future. If destruction introduces await points, the destructor future must be included in auto-trait reasoning.

## 15. Migration strategy

Stage 1: compiler machinery. Teach the compiler to model async destruction internally and generate async drop glue where needed.

Stage 2: stable async cleanup for synchronously droppable types. Let libraries provide better cleanup in async contexts while retaining existing synchronous behavior.

Stage 3: diagnostics and lints. Warn on fragile explicit cleanup patterns, ignored cleanup futures, and types that appear to want async-only destruction.

Stage 4: opt-in async-only destruction. Allow selected types to declare that synchronous destruction is not available.

Stage 5: future edition improvements. Consider stronger defaults, clearer bounds, or syntax improvements once experience accumulates.

## 16. Open questions

See [open-questions.md](open-questions.md).

## 17. Conclusion

The proposed end-state is:

- synchronous code requires synchronous destruction
- async code can perform async destruction
- some types may be async-only destructible
- implicit async cleanup should be reliable and cancellation-shielded
- dynamic async destruction should avoid mandatory extra allocation

This model tries to make async cleanup part of Rust's ownership story while keeping the design incremental and honest about unresolved questions.
