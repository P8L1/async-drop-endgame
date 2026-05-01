
# Open Questions

## 1. Synchronously undroppable types

- What is the right surface spelling for a type that cannot be synchronously destroyed?
- Should sync destructibility be represented as an auto trait, a lang item, a marker trait, or something else?
- How should generic bounds express "may be async-only destructible" without making common signatures noisy?
- What diagnostics are needed when an async-only value is moved into a sync context?
- Can the model distinguish "sync drop is impossible" from "sync drop is possible but loses quality of cleanup"?

## 2. Cancellation shielding

- What exactly does cancellation shielding mean for compiler-generated async destruction?
- Can implicit async destruction be cancelled by panic unwinding, task abort, executor shutdown, or process exit?
- Should users be able to opt out of shielding for specific types or scopes?
- How should shielding interact with timeout wrappers around an async function?
- What diagnostics can explain hidden cancellation-shielded await points?

## 3. Explicit async drop

- Should there be a way to explicitly trigger async destruction before scope exit?
- If explicit async destruction exists, should it consume the value or pin-project into it?
- How should explicit async cleanup interact with implicit async destruction at scope exit?
- Should calling `.close().await` suppress later `AsyncDrop`, or should it merely change the state that `AsyncDrop` observes?

## 4. `Drop` plus `AsyncDrop`

- Should a type be allowed to implement both `Drop` and `AsyncDrop`?
- If both exist, should async contexts run only `AsyncDrop`, or should `Drop` also run as part of field destruction?
- How can libraries express a synchronous fallback that is safe but less complete than async cleanup?
- What rules prevent double cleanup when both traits are present?

## 5. Dynamic dispatch

- What vtable information is needed to async-destroy a `dyn Trait` value?
- Can dynamic async destruction avoid mandatory allocation for common trait object representations?
- Is allocation acceptable for some erased representations if it is not required by the language model?
- How should object safety rules mention async destructors?
- How should async destruction work for `Box<dyn Trait>`, `Pin<Box<dyn Trait>>`, and custom pointer types?

## 6. Auto traits

- How should `Send`, `Sync`, `Unpin`, and related auto traits account for async destructor futures?
- If an async destructor is not `Send`, when does that make the containing async function future not `Send`?
- Can diagnostics point directly to the destructor that affects auto-trait inference?
- Are additional marker traits needed for cancellation shielding or async destructibility?

## 7. Pin projection

- What projection guarantees does `AsyncDrop` need for pinned fields?
- Should the language provide special projection support during async destruction?
- How should unsafe code reason about partially destroyed pinned values?
- Can existing pin-projection library patterns migrate cleanly?

## 8. Destruction order

- How exactly should field destruction order map from synchronous drop glue to async drop glue?
- What happens when one field's async destructor awaits while other fields remain alive?
- How should partially initialized values be async-destroyed after errors?
- How should panic during synchronous fallback interact with async field destruction?

## 9. Migration

- Which parts can be stabilized first without committing to async-only destructibility?
- What lints would give useful signal without overwhelming existing code?
- How should standard library types adopt async cleanup, if at all?
- What edition changes would make the final model easier to teach?

## 10. Minimal viable stabilization

- What is the smallest stable feature that gives libraries real value?
- Can `AsyncDrop` for synchronously droppable types stabilize before `!Destruct` or equivalent syntax?
- Is compiler-generated async drop glue useful without dynamic dispatch support?
- Which unresolved questions must be answered before any public API is exposed?
