# Cached one-shot systems

**Bevy 0.15** introduces a convenient new "cached" API for running one-shot systems:

```rust
// Old, uncached API:
let foo_id = commands.register_system(foo);
commands.run_system(foo_id);

// New, cached API:
commands.run_system_cached(foo);
```

To better understand why this new API is useful, let's take a look at what systems really are behind the curtains:

A system in Bevy is a function together with its _system state_, which is some data that helps manage and run the system efficiently. As of 0.15, each system state takes up 240 bytes of space, plus ~0-1000 bytes for system parameter state: 56 for `Commands`, 16 per `Res` / `ResMut`, 216 per `Query` (plus 8 per component accessed or filtered by the query), and so on. Registering a one-shot system spawns a new `RegisteredSystem` entity with space for this system state and returns the `Entity` as a `SystemId`. The system state is then initialized the first time the system is run.

So, for example, suppose you write some code that registers and runs a system whenever a button is pressed. Unfortunately, you now have a memory leak: whenever the button is pressed, a new copy of the system will spawn with its own system state. A common pattern that fixes this leak is to cache the `SystemId` in a `Local` or `Resource` and then only register the system the first time the button is pressed. This is exactly what the new "cached" API does for you! When you `run_system_cached`, the `SystemId` will be saved in a generic `CachedSystemId<S: System>` resource to be reused later.

You can see this in action if you register the same system multiple times and compare the returned IDs:

```rust
// Uncached API:
let id1 = world.register_system(quux);
let id2 = world.register_system(quux);
assert!(id1 != id2);

// Cached API:
let id1 = world.register_system_cached(quux);
let id2 = world.register_system_cached(quux);
assert!(id1 == id2);
```

### Comparison to `run_system_once`

There were two ways to run one-shot systems, and now there are three. The approach taken by `run_system_once` is to set up a system, run it once, and then tear it down. This means that `run_system_once` _does_ avoid the memory leak problem. However, any system parameters like `Local` and `EventReader` that rely on persistent state between runs will be broken; and any system parameters like `Query` that rely on cached computations to improve performance will have to rebuild their cache each time, which can be costly. As a consequence, `run_system_once` is only recommended for diagnostic use (e.g. unit tests), and `run_system` or `run_system_cached` should be preferred for "real" code.

### Limitations

The cached API can only work if it can guarantee that different systems are never cached under the same `CachedSystemId<S>`. In other words, there should be no more than 1 distinct system of type `S`. This is true when `size_of::<S>() == 0`, which is almost always true in practice. Thus, to enforce correctness, the new API will give you a compile-time error if you try to use a non-zero-sized function (like a function pointer or a capturing closure).
