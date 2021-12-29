# Part 2. Why *all* message queues suck

## Switching message queue

After much effort, we port our application to use a "better" message
queue. Something like Kafka or Apache Pulsar. We have more control
over how messages are distributed to workers which helps us eliminate
the high latencies which customers were seeing, and we might be able
to partition tasks in such a way that potentially conflicting tasks
are processed sequentially rather than in parallel, which helps
with the database contention.

*If we're using Kafka, we might be able to get rid of our "outbox"
entirely, by storing queue offsets directly in our database, but
this brings its own challenges.*

We modify the retry logic in our "outbox watcher": now we remove
tasks from the outbox as soon as our message queue accepts them, and
rely on the retry logic within the message queue to handle failures
from running the task.

This allows us to reduce some of the load on our "outbox" table,
since it will remain relatively small at all times. Our database
can likely keep it entirely in memory. The downside to this change
is that we lose the ability to cleanly restore our application
state from a backup, and we need our persistent volume back...

Deployment of our application is now significantly more complicated
than it once was.

## Complexity

The new message queue is not a panacea. There are lots of little gotchas
here and there, but at least they seem solvable. Somebody somewhere will
have run into them before, and over time we get closer to a reliable and
stable system.

The problem is the additional complexity this has introduced to our
codebase and deployment. We end up with lots of rules for what should
and should not be done to avoid running into problems. A lot of code is
dedicated to handling various failure modes, and our business logic has
become harder to isolate and test independently.

Everywhere we trigger a background job, we have to add extra
columns to record the ID of that job so that when it completes, we
can match its result back up with the correct entity, and this
all needs to be mocked out when under test.

Finally, the routing and partitioning logic in our message queue is
intricately tied to our database access patterns and updates to either
one can easily result in unexpected performance regressions.

We have something that *works*. Eventually it might even *work well*. But
we're a long way from our nice simple CRUD architecture.

## Why is this so hard?

People have been building these kinds of applications for a very long
time. Why are the tools still so hard to use? Why do they fit together
so poorly? I've thought about this for a long time, and I believe the
fundamental reason for this complexity is the lack of a shared transactor.

A transactor is a process that ensures transactions are either fully
committed or fully rolled back. It is a fundamental building block to
writing correct applications, but it typically operates at a very low
level.

A SQL database will have its own transactor. A message queue will have
its own transactor (although it may not call it that). Most systems
which store persistent state will have some kind of transactor to
ensure correctness, but no such tool is available to the application
itself.

Every transactor has a domain in which it operates, and the boundary
between two transactors is a place where failures can and will occur.
Any time there is communication between domains owned by different
transactors, extra complexity is needed:

- Automatic retries with backoff.
- Handling of failures and timeouts.

These things cannot be handled "generically". A true failure here (i.e.
one that cannot be retried) is something that eventually needs to be
surfaced to a user somewhere. What that actually means is intrinsically
tied to the business logic of the application.

Furthermore, the persistent store beneath a particular transactor
might not support the functionality we need to implement this extra
error handling efficiently. Take a SQL database for example:

- Pros:
    - Can easily store the "outbox" in a table.
- Cons:
    - Frequent updates to the "outbox" table are a bottleneck.
    - "Watching" the "outbox" table for new rows may require polling.

On the other hand, a message queue has this retry logic built in, but
if we want to e.g. find out how many tasks are "stuck", or get any
insight into the state of the message queue that requires a
non-trivial query, then the message queue likely won't support that.

Wouldn't things be a lot easier if we could just have all of our
persistent state live under the same transactor? We'd still have
to worry about failure modes when talking to external systems, but
at least *for our own code, and our own business logic* we can just
assume reliable communication everywhere.

## FoundationDB

Unsurprisingly, I'm not the first person to think this way. FoundationDB
is a distributed key/value store which is intended to be used as the
*foundation* for other tools which need to store persistent state.

In FoundationDB, an application will typically talk to higher level
"layers" which run on top of FoundationDB. These layers could include
a SQL database, a message queue, etc. All the layers use the same
transactor, so you can easily update a row and send a message, all in
one atomic transaction. Furthermore, you can
backup and restore the entire FoundationDB's state in one go.

The premise here is that useful layers can be built upon a
shared key/value store, and that sharing a transactor is useful.

## What can we do with a tool like FoundationDB?

Obviously we could mimic our old architecture, with a SQL layer and
a message queue layer, but FoundationDB opens up a lot of other
options too.

I've been pursuing one of these ideas with a project I'm calling
AgentDB. This project borrows a lot of ideas from actor systems,
but with a strong focus on eliminating code which is not required
for the business logic itself.

This will be the focus of [part 3](../part3)!
