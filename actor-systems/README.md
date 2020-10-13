# Building an async-compatible actor system

I [previously wrote](../async-mutexes/README.md) about the problems with
implementing an async-aware `Mutex` type in Rust, and showed how
treating a mutex as a task to be spawned can solve many of those
problems.

This time, I want to talk about my experiences building an actor
system out of that primitive mutex concept.

## What is an actor?

Real actor systems encompass a huge amount of functionality, but there
are certain properties of an actor that are fundamental, and common
to all actor systems.

An actor:

- Has state.
- Receives messages.
- May modify its state in response to a message.
- May send messages to other actors.
- Has exclusive access to its own state, ie. no other code can
  directly access that state.

## Why are actors useful?

Again, I'm going to limit this to the absolute fundamentals of
what an actor system can do: with real actor systems there are
many additional advantages that may be specific to that system.

- As an alternative to a mutex, an actor can provide higher
  throughput.

  When a mutex is unlocked, the next thread must be
  woken up, a context switch must occur, and then the next thread
  must take the lock. During that time, no useful work can be done
  on the data protected by the mutex. As the amount of contention for
  the mutex increases, the overhead of this synchronization also
  increases, reducing throughput further.

  In contrast, with an actor, the synchronization cost does not eat
  into the actor's productive time: the actor has exclusive access to
  its own state at all times, and will process messages as fast as it
  can. With message passing, the synchronization cost is both reduced,
  and largely paid by the message sender instead of the receiver.
  Furthermore, the maximum throughput of the actor stays constant: it
  is not affected by contention in the way a mutex is.

- Since all access to an actor's state goes through the actor, it can be
  easier to reason about than code using mutexes. Even if multiple
  threads are sending messages to the actor, the actor itself
  processes messages one at a time.

  In this way, an actor can be easily modelled as a state machine.
  Alternatively, a protocol may be specified via state-machines, and
  this protocol specification can be almost mechanically transformed into
  the logic for one or more actors.

  When the code implementing a specification so closely matches the
  specification itself, it's much easier to verify that the
  implementation is correct.

  (See also, [CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes))

- When writing code with a lot of concurrency, using actors can
  dramatically simplify the code.

  Actors are a tool that a programmer can use to turn a huge concurrent
  problem into a large number of single-threaded problems.

  They enable individual parts of the system to be changed without
  having to re-evaluate the big picture.

  In contrast, attempting to solve complex concurrency problems using
  only primitive tools can result in monolithic programs where minor
  changes can have far-reaching repurcussions.

## What's wrong with existing actor systems for Rust?

Now that you're convinced of why actor systems are a good idea, it's
probably worth taking a look at the existing options.

### [Actix](https://crates.io/crates/actix)

Actix is the most well-known actor system for Rust, and one of the more
mature. It is very fast, and a lot of effort has been expended in optimizing
it. The area where I find it most lacking is in ergonomics.

For me at least, the most compelling reason to use an actor system is to
make it easier to solve problems. Yet there are several design choices
made by Actix which, in my opinion, work against this goal:

1. Actix has its own `ActorFuture` trait which is not compatible with
   Rust's built-in `Future` trait.

   The difference between these two traits is in the `poll` method:

   `Future`:

   ```rust
   fn poll(
       self: Pin<&mut Self>,
       context: &mut Context
   ) -> Poll<Self::Output>
   ```

   `ActorFuture`:

   ```rust
   fn poll(
       self: Pin<&mut Self>,
       state: &mut Self::Actor,
       actor_ctx: &mut <Self::Actor as Actor>::Context,
       context: &mut Context<'_>
   ) -> Poll<Self::Output>
   ```

   This extra `state` parameter is how an actor is able to modify its
   own state when responding to a message. Likewise, the `actor_ctx`
   parameter provides things like the actor's own address, for use when
   sending out messages to other actors.

   You can adapt a `Future` for an API expecting an `ActorFuture`, but not vice-versa.

2. As a result of having an incompatible `Future` trait, actix is also
   incompatible with Rust's async/await syntax. (Unless you use an
   interop crate like [`actix-interop`](https://crates.io/crates/actix-interop)
   &lt;/shameless-plug&gt;)

3. Actix allows multiple asynchronous message handlers to run
   concurrently. Because `state` is only borrowed for the poll
   method, actix is able to interleave calls to `poll` to multiple
   futures executing on the actor.

   I can see uses for this, but making it the default can lead to
   surprising results, as it breaks the assumption that a given
   message handler has exclusive access to the actor's state for the
   duration of its execution.

   Luckily actix _does_ have a way to run a handler with exclusive
   access, but it must be opted into, and comes with its own ergonomic
   cost. Even then, you cannot borrow the actor's state across yield
   points, so methods with the following signatures cannot
   be called on the actor's state:

   ```rust
   // `.await`ing these methods would result in `self` being borrowed
   // across a yield point, which is not allowed.
   async fn ref_receiver(&self, ...);
   async fn mut_receiver(&mut self, ...);
   ```

   As you can imagine, this is extremely limiting, and leads to some
   very ugly work-arounds.

4. So much boilerplate!

   - You must define a type for each message.
   - You must implement the `Message` trait for each message.
   - You must define a "response" type for each message.
   - You must implement the `Handler<M>` trait for each message
     an actor can handle.
   - You must implement the `Actor` trait for each actor.

   Each piece makes sense, but in total there's a huge barrier to even
   getting off the ground, and it detracts from thinking about the
   problem you're actually trying to solve.

### [Riker](https://crates.io/crates/riker)

Riker is a little more opinionated than `actix`, which allows it to
avoid some of the boilerplate (for example, the `Handler` and `Actor`
traits are unified). It also seems to be quite mature with support for many
more advanced actor system concepts.

However, it does not appear to be fully compatible with futures: whilst
it is possible to spawn futures onto the system, message handlers are
required to be synchronous themselves. This may be a good trade-off
for many applications, but it rules it out for me as a truly async-compatible
actor framework.

### [Acteur](https://crates.io/crates/acteur)

Of the actor systems I've reviewed, `acteur` is probably the closest
to meeting all my requirements for an actor system. It is fully
async/await compatible, and I only have a few criticisms:

- It doesn't seem to provide a way to run futures "in the background"
  on an actor: this is useful for tasks that do not require
  access to the actor's state, but should stop when the actor
  is stopped.

  One example of this would be sending a message to the actor after a
  certain amount of time has passed. You would still want the actor
  to be able to handle other messages during that time interval.

- It's locked into the `async-std` runtime.

- There's still a decent amount of boilerplate necessary to get an
  actor up and running, and certain concepts like activation
  and deactivation are baked-in, which many applications will not
  care about.

### Others

There are many more actor systems in Rust, forks of actor systems I've
already mentioned, etc. but I haven't had time to review all of them
in detail.

## What _could_ an ergonomic Rust actor system look like?

I want:

1. It to be fully async/await compatible. Mashing together
   future combinators is so last year!

2. To be able to call async methods that borrow the actor state.

3. Each message handler to run to completion before moving onto
   the next message (at least by default).

4. To be able to run "background" futures on an actor that don't need
   access to its state.

5. To have a zero-boilerplate way to define an actor. I don't want a
   separate trait implementation for every single message I can
   handle.

6. To support static and dynamic polymorphism using normal traits.

7. To make no &mdash; or at least minimal &mdash; use of unsafe code.

8. To achieve all of the above with the least amount of "magic" possible.
   Actor code should still be normal Rust as far as the user is concerned.

Putting all that together, we end up with something like this:

```rust
#[derive(Default)]
struct AutomaticSupportActor {
    // The actor state
    computer: Computer,
    problems_fixed: u32,
}

impl Actor for AutomaticSupportActor {}

#[async_trait]
trait SupportTechnician: Actor {
    async fn fix_problem(&mut self);
}

#[async_trait]
impl SupportTechnician for AutomaticSupportActor {
    async fn fix_problem(&mut self) {
        self.computer.turn_off().await;
        self.computer.turn_on().await;
        self.problems_fixed += 1;
    }
}

fn example() {
    let x: Addr<dyn SupportTechnician> = spawn(MyActor::default());

    // This must be a macro, because we want the method call to
    // happen within the execution context of the actor.
    send!(x.fix_problem());
}
```

We lean heavily on standard tools such as the
[async-trait](https://crates.io/crates/async-trait)
crate, which should be familiar to anyone who's done much async programming in Rust
already.

## How do we actually implement that?

This is the hard part! Spoiler: to get this to run on stable Rust, we
will need to make a few minor compromises.

Looking at the code above, we need at least 3 things:

1. A way to spawn an actor (`spawn`).
2. A type to store the address of an actor or trait object (`Addr<T>`).
3. A macro to call a method on an actor.
4. A way to convert the address of a concrete actor type to a trait object (upcast it).

In addition, although not necessary in the above code, we'll want:

1. A way for an actor to get its own address.
2. A way for the caller to wait on a response from an actor.
3. A way for an actor to spawn background futures.
4. A flexible way to manage the lifetime of an actor.

## Implementing `send!`

We need an ergonomic way to send functions to be executed.

The first hurdle is that the type signature of async methods which borrow one
of their arguments is complex:

```rust
for<'a, T> FnOnce(&'a mut T) -> impl Future<'a, T>
```

To make things worse, it is not currently possible to write a closure with
this exact signature due to language limitations.

I tried a number of ways to work around this, but in the end the only option
which did not come with too many downsides was to use a macro, and to box
things whenever the types got too out of hand.

The macro expands to something like this:

```rust
addr.send_mut(Box::new(move |x| {
    (async move {
        x.method(arg1, arg2).await
    }).boxed()
}));
```

By boxing the return type of the closure, we transform the signature to:

```rust
for<'a, T> FnOnce(&'a mut T) -> BoxFuture<'a, T>
```

This signature _can_ be achieved with the closure above.

In total we require two allocations: one to erase the type of the closure to be
executed on the actor, and a second to erase the type of the future it
returns.

## Spawning an actor

We want to implement a task that pulls an "action" from a channel, and then
polls it to completion before pulling the next "action".

We'll define an "action" as a function that operates on the actor state
and returns a future. Keep in mind: the future it returns will still
be borrowing the actor state, so we need to be careful around lifetimes.

The future will output a `bool` indicating whether the actor should stop.

```rust
type MutItem<T> = Box<dyn for<'a> FnOnce(&'a mut T) -> BoxFuture<'a, bool> + Send>;

async fn actor_task<T>(
    mut value: T,
    mut mut_channel: mpsc::UnboundedReceiver<MutItem<T>>,
) {
    loop {
        // Obtain an item
        let current_item = if let Some(item) = mut_channel.next().await {
            item
        } else {
            return
        };

        // Call the function, and then poll the future it returns to completion
        let current_future = current_item(&mut value);
        if current_future.await {
            return;
        }
    }
}
```

To spawn an actor, we just spawn this async task, passing in the actor state.

However, we'll also want the actor to be able to execute background futures
which don't require access to the actor state. We can achieve this using the
`select_biased!` macro and the `FuturesUnordered` type:

```rust
type MutItem<T> = Box<dyn for<'a> FnOnce(&'a mut T) -> BoxFuture<'a, bool> + Send>;
type FutItem = BoxFuture<'static, ()>;

async fn actor_task<T>(
    mut value: T,
    mut mut_channel: mpsc::UnboundedReceiver<MutItem<T>>,
    mut fut_channel: mpsc::UnboundedReceiver<FutItem>,
) {
    let mut futs = FuturesUnordered::new();
    loop {
        // Obtain an item
        let current_item = loop {
            if select_biased! {
                _ = futs.select_next_some() => false,
                item = mut_channel.next() => if let Some(item) = item {
                    break item
                } else {
                    true
                },
                item = fut_channel.select_next_some() => {
                    futs.push(item);
                    false
                },
                complete => true,
            } {
                return;
            }
        };

        // Wait for the current item to run
        let mut current_future = current_item(&mut value).fuse();
        loop {
            select_biased! {
                done = current_future => if done {
                    return;
                } else {
                    break
                },
                _ = futs.select_next_some() => {},
                item = fut_channel.select_next_some() => futs.push(item),
            }
        }
    }
}
```

We use `select_biased!` rather than `select!` as we don't need to be fair: in
fact, if anything, we'd prefer to give priority to the `current_future` which
mutably borrows the actor state, and that's exactly what we do.

## Storing the address of an actor

The actor address just needs to store the two channels used for sending
items to the actor for execution.

```rust
struct AddrInner<T> {
    mut_channel: mpsc::UnboundedSender<MutItem<T>>,
    fut_channel: mpsc::UnboundedSender<FutItem>,
}
```

However, I wanted to allow actors to automatically stop when there are no
more references, and for that we need to use reference counting.

This is simple enough: we just wrap our `AddrInner` in an `Arc` or `Weak`
depending on if we want to keep the actor alive or not.

```rust
struct Addr<T> {
    inner: Arc<AddrInner<T>>,
}
```

## Supporting trait objects

This is perhaps the hardest requirement: `CoerceUnsized` and friends are
all unstable so we can't make this conversion be completely implicit, and
still support stable Rust.

I found the easiest way was to define a macro, `upcast!(...)` to do
this conversion. It expands to a simple method call `.upcast(|x| x as _)`
and relies on type inference to work correctly.

The second part is even harder: how do we make `Addr<T>` also work as
`Addr<dyn Trait>`? We can't use specialization or anything like that,
and the Rust type system starts to feel very limited...

To solve this, I essentially ended up building a manual vtable:

```rust
pub struct Addr<T: ?Sized + 'static> {
    inner: Option<Arc<dyn Any + Send + Sync>>,
    send_mut: &'static (dyn Fn(&Arc<dyn Any + Send + Sync>, MutItem<T>) + Send + Sync),
    send_fut: &'static (dyn Fn(&Arc<dyn Any + Send + Sync>, FutItem) + Send + Sync),
}
```

We erase the concrete type of the actor on construction via the
`Arc<dyn Any + ...>`, and we store the address of some static methods to
send items and futures to the actor.

When we `upcast!` an actor, we leave the `inner` value unchanged, but
re-assign `send_mut` to a different "glue" function that performs the required
conversion of items sent to the actor.

Couple of things to note: this is all possible to implement using zero unsafe
code, and only a single additional allocation is required, which happens
when sending a message to an `Addr<dyn Trait>` in order for the "glue code"
to be injected.

In practice, a single line of `unsafe` code _is_ used for performance
reasons, but it will be possible to remove this if
[my PR to Rust](https://github.com/rust-lang/rust/pull/77688) is merged.

## Actor trait

We want to allow an actor to access its own address, and also to implement
some standardized error handling. To that end, we add two methods to the
`Actor` trait:

```rust
#[async_trait]
pub trait Actor: Send + 'static {
    /// Called automatically when an actor is started. Actors can use this
    /// to store their own address for future use.
    async fn started(&mut self, _addr: Addr<Self>) -> ActorResult<()>
    where
        Self: Sized,
    {
        Ok(())
    }

    /// Called when any actor method returns an error. If this method
    /// returns `true`, the actor will stop.
    /// The default implementation logs the error using the `log` crate
    /// and then stops the actor.
    async fn error(&mut self, error: ActorError) -> bool {
        error!("{}", error);
        true
    }
}
```

We avoid boilerplate by providing reasonable default implementations.

## Returning values

We can allow callers to access the return value of actor methods by adding
a `call!` macro to completement our `send!` macro. It will be implemented
the same way, except that it will create a `oneshot::channel` to send back
the return value from the actor method.

The caller can `await` this channel to get their result.

## All the trimmings

What we've got so far is completely executor agnostic, but is pretty much
the bare minimum to be considered an actor system.

To make life easier for users I have added a couple of standard integrations
behind feature flags, for the two main async runtimes:

- `tokio`
- `async-std`

These modules provide runtime-specific `spawn_actor` functions, and abstract
across timer implementations in the two runtimes.

I also built a generic `timer` module which uses these abstractions to allow
actors to easily set and clear timers. Timers in an actor system come with
their own small set of challenges: if a timer works by firing messages at
the actor, it is not possible to reliably "cancel" a timer, as the message
may have already been sent.

The `timer` module solves this by allowing spurious timer messages, and
requiring the user to check that the timer has actually elapsed before
responding to a `tick` event.

## That's it!

This covers the core of how `act-zero 0.3` is implemented.

This is the second major iteration of the design, and I'm really happy with
how it came out. You can see the real code and some example uses of the crate
in the [github repo](https://github.com/Diggsey/act-zero).

In order to test the design of `act-zero` I've been building an implementation
of the `raft` distributed consensus algorithm, `raft-zero`. This has helped
me stay focused on building an actor system that is ergonomic, and designed
to solve problems rather than create them!
