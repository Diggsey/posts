# Part 1. Why *simple* message queues suck

## The lifecycle of a web application

Web applications typically start their life as simple CRUD applications.
They receive a request, run some queries against a database and finally
return a response. When there are no requests, the database state
remains unchanged.

Our deployment might look something like this:
1. SQL database
    - Tables map 1:1 with business concepts.
2. Stateless web server running our code.

This is a great architecture:
- It's simple and well understood.
- Database transactions solve most consistency/atomicity requirements.
- It can scale to large numbers of clients, either through the use
  of a distributed database, or effective use of caching in read-heavy
  workloads.
- All of our state is contained in the SQL database, so backup and
  restore is very easy.

The problems begin when we inevitably need to do something that doesn't
fit into this model. We might need to:

- Make a request to a slow external service.
- Run some background work after a request finished.
- Perform a task periodically.
- Run some code reactively (eg. when this row in the database changes,
  do this other thing).

At this point, the obvious tool to reach for is some kind of message
queue. This is where things go downhill fast...

## Adding a message queue

*I'm going to pick on RabbitMQ here, but only because that's what I
have most experience with. There are many products in this space
that suffer from largely the same issues.*

We add RabbitMQ to our deployment:

1. SQL database
    - Tables map 1:1 with business concepts.
2. Stateless web server running our server code.
3. **RabbitMQ broker.**
4. **RabbitMQ worker running our background tasks.**

We configure RabbitMQ to send ACKs on task completion, so that tasks
are automatically retried if they fail.

## Message queue persistence

Everything is running smoothly, until one day we get a spike in load.
Shortly thereafter, we start getting various bug reports from customers
which all seem to relate to background tasks not running.

Upon checking the logs we realize what happened: with the spike in load,
the RabbitMQ broker built up a backlog of messages, and ended up being
OOM killed. It was automatically restarted by its supervisor, but we
lost all the messages it had in memory at the time.

We spend many hours of developer time figuring out how to rerun the
lost tasks, but in the end this was our fault for not giving RabbitMQ
somewhere to persist messages reliably:

1. SQL database
    - Tables map 1:1 with business concepts.
2. Stateless web server running our server code.
3. RabbitMQ broker.
4. RabbitMQ worker running our background tasks.
5. **Persistent volume for RabbitMQ broker.**

## Backup and restore

We're on our day off, when suddenly we get paged. Our application is
returning errors on all requests. It looks like there was a hardware
failure, and the database state has become corrupted!

No problem; we have backups for a reason, and we spin up a fresh
database instance and restore into it. We even have point-in-time
restore, so we can restore to right before the hardware failure.

However, even after restoring, we see some strange errors coming in.
Backgrounds tasks trying to access records in the database that don't
exist. Background tasks that seem to have been lost altogether. Of
course! We restored the database state, but we didn't touch the state
of the message queue, and now they're out of sync.

We spend many more hours of developer time reconciling this inconsistent
state, and begin considering how we might address this issue. Hardware
failures are rare, but there are lots of reasons we might need to rely
on our backups, and this reconciliation process is unpleasant and
error prone...

## Non-transactional transactions

Business is going well, and we're seeing a lot more load as a result.
For some reason, our task queue seems to be getting less and less
reliable. When the load goes up by a fixed amount, the rate of
duplicate or lost tasks seems to double! What used to *never*
happen is now a daily occurrence, and a significant amount of
developer time is spent trying to reconcile inconsistencies as a result.

After yet more investigation, we find that the duplicate tasks occur
when a transaction is rolled back due to a conflict. In that case, we
retry, and this results in duplicate tasks being scheduled.

The missing tasks seem to happen when we schedule tasks *after*
committing the transaction. It looks like there's a brief window
where a network blip or other intermittent error can result in the
transaction being committed and the task never being scheduled.

Surely this problem has been solved before?

## The outbox pattern

Sure enough, our research yields results. It looks like there's a
simple pattern to solve this: we simply write any tasks we want to
run to a database table, and then have a separate process that takes
things from this "outbox" and sends them to the task queue.

Once a task completes, it removes itself from the table, ensuring that
it won't be retried.

1. SQL database
    - Tables map 1:1 with business concepts.
    - **Outbox table.**
2. Stateless web server running our server code.
3. RabbitMQ broker.
4. RabbitMQ worker running our background tasks.
5. Persistent volume for RabbitMQ broker.
6. **Outbox watcher.**

This new "outbox watcher" takes rows from the "outbox" table and
schedules them in RabbitMQ. If we're lucky, our database has some
means to do this efficiently, but more than likely it does not.
In that case, we might have to resort to polling the table.

## The retry-pocalypse

Another day, another developer deploys some buggy code. But this time,
instead of causing a few tasks to error, the entire system implodes!
Messages are building up in the task queue at an absurd rate, and
the workers are maxed out.

We roll back the change, but it still results in a significant outage.
What's going on? We have exponential backoff on the tasks, so a few
failing tasks shouldn't cause this much of an issue.

The problem is that we now have two retry mechanisms at play: there's
RabbitMQ's retry mechanism, and there's also the retries triggered
by the "outbox" watcher. We need the latter, so we decide to turn off
the former by sending an ACK as soon as a message is received by a
worker.

On the plus side we can also get rid of our persistent volume for
RabbitMQ!

1. SQL database
    - Tables map 1:1 with business concepts.
    - Outbox table.
2. Stateless web server running our server code.
3. RabbitMQ broker.
4. RabbitMQ worker running our background tasks.
5. **~~Persistent volume for RabbitMQ broker.~~**
6. Outbox watcher.

## Performance problems

The business continues to perform well, but the same can't be said
of our web application. Customers are starting to complain about tasks
running slowly, and on top of that, our "outbox" table is becoming a
bottleneck with our database regularly hitting CPU and I/O limits.

*If we were using MySQL, we might find that we have to keep several
hundred "dummy" rows in the "outbox" table, or else the gap locks
used by MySQL cause all our transactions to conflict with each other!*

We move our database to a beefier machine, and look nervously at the
projected future loads... It seems to have at least temporarily solved
the database load issue. However, customers are still complaining about
slow tasks.

We add some more metrics, and find that a lot of time is spent between
a message being scheduled and it being picked up by a worker. What's
going on? We just scaled up our workers and they're mostly idle. Why
are they not picking up new tasks? We try disabling "pre-fetch" in
the workers, but it doesn't seem to help.

Many *many* hours of investigation later, the problem becomes clear.
RabbitMQ uses the AMQP protocol to deliver messages. This is a "push"
based protocol, meaning that the broker pushes messages to workers,
rather than workers "pull"ing them from the broker when they're ready.

How does the broker decide when to send a new message to a worker?
Well, assuming pre-fetch is disabled, it sends a new message when the
worker ACKs the previous one. When we updated our workers to send ACKs
early to avoid the retry-pocalypse, we inadvertently made it so that
new tasks could get stuck behind other slow tasks!

Unfortunately, there doesn't appear to be a way to solve this, as it's
intrinsic to the underlying protocol. We try to work around it by
splitting tasks into those we expect to run quickly, and those which
might take a little more time, scheduling the different types onto
different queues. We split our workers up similarly, so that some
workers are dedicated specifally to each latency group.

We've complicated our deployment yet again, and it hasn't completely
eliminated the problem, but at least customers seem to be less vocal
about it...

## Transaction conflicts

We've noticed the number of transaction rollbacks caused by conflicts
has steadily been increasing. We're scheduling background tasks in
response to certain updates in the database, and this is causing
bursts of writes to the same set of rows in the database. The background
tasks have a much higher than expected chance of conflicting with
concurrent requests and other tasks.

We try to mitigate this by explicitly locking rows we're going to update
at the beginning of the transaction, so that other potentially conflicting
transactions will wait rather than do redundant work that must later be
reverted.

This gets rid of the transaction conflicts, but as a result, and combined
with our segregation by latencies, we're making very inefficient use of
our workers, with lots of them idle, either waiting on database locks, or
simply belonging to a latency group that is currently idle.

## The grass is always greener

At this point, we're getting pretty fed up. Those promises of simplicity
and performance when we first went looking for a task queue seem
laughable now.

Perhaps it was a mistake to shy away from the more advanced systems like
Kafka or Apache Pulsar? Maybe the designers of those systems have spent
more than 5 seconds thinking about how their product might actually
integrate into a larger application?

Continued in [part 2](../part2)...
