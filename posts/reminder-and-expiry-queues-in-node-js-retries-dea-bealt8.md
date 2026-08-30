# Reminder and expiry queues in Node.js: retries, dead letter queues, idempotent consumers

Use your primary database as the queue until it stops being enough. For a one-person game backend that has to expire a ten-minute hold on a limited-edition skin, mail the player a reminder before the hold lapses, and survive email send failures from a provider you don't control, a `task` table with `FOR UPDATE SKIP LOCKED`, an idempotent consumer and a dead letter state covers the whole job — about eighty lines of Node.js. The harder question is not which broker to adopt. It's how late an expiry is allowed to be, and what you're willing to pay per month for that number.

Latency is the price knob. Most of us set it far tighter than any player can perceive.

## What a ten-minute hold actually promises

A reservation is a promise with a deadline attached: for ten minutes the item is yours, and after that it returns to the pool. Two parties depend on that deadline. The storefront needs the inventory count to be honest, because a hold that expires 40 seconds late shows "sold out" for an item nobody owns — and during a drop, 40 seconds of phantom scarcity is real lost revenue. The player needs the reminder to arrive before the deadline, not after it, which makes the reminder job a scheduled send with a hard upper bound rather than a best-effort notification.

So the design axis isn't throughput. It's how much lateness you're buying out of, and at what fixed monthly cost.

| Lateness budget for an expired hold | Mechanism that fits | What you pay for it |
|---|---|---|
| Minutes | One sweep per minute over an indexed `run_at` column | A single query per minute; effectively free |
| 1–5 seconds | Workers polling with `FOR UPDATE SKIP LOCKED` | Constant wakeups you pay for at 3 a.m. too |
| Under 200 ms | Pushed delivery: delayed messages in a broker, or in-process timers rebuilt from the table on boot | A broker to operate, plus its own failure modes |

A worker asking "anything due?" every second runs about 86,400 queries a day per worker, and nearly all of them return zero rows. That's cheap on a database you already run and absurd as a reason to introduce a second stateful system. Pick the loosest lateness budget the product can defend, then buy exactly that much and no more.

## Which queue should I use for user reminders in Node.js, and how do email send failures get handled?

Those are two different problems that get merged into one shopping decision, which is why the answer usually comes out wrong.

The first problem is durable scheduling: some row says "at 14:32:10, release reservation 881 and stop tracking it." That state belongs next to the invariant it protects. If the hold lives in Postgres and the expiry lives in a broker, you now own a distributed transaction between them, and the failure you'll actually hit is a released hold with no matching inventory row — or the reverse, which is worse, because it's silent.

The second problem is delivery to a third party. Your email provider will rate limit you during a drop, and the correct reaction to HTTP 429 is to read `Retry-After` and schedule the retry for exactly that long, not to spin an exponential backoff you invented. Treat provider responses in three buckets: permanent rejections (a malformed address) go to the dead letter state on the first attempt, transient rejections retry with jitter, and rate limits retry at the time the provider named. Same logic for outbound webhooks to the game server, with one addition — signature and payload must be identical across attempts, or the receiver's own dedupe key changes and you've fanned out a duplicate.

Brokers are genuinely better once fan-out gets wide: thousands of consumers, cross-language workers, or a lateness budget in milliseconds. Under that, a table is less code, less to operate, and it's queryable during an incident, which matters more than it sounds at 2 a.m.

## The smallest thing that works

The consumer claims a batch, does the work, and writes back a terminal state. `SKIP LOCKED` is what makes concurrent workers safe without a lease table; rows already locked by another transaction are stepped over instead of blocking.

```ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

type Task = {
  id: string;
  kind: "hold_reminder" | "hold_release";
  reservationId: string;
  attempts: number;
};

async function claim(limit = 20): Promise<Task[]> {
  const { rows } = await pool.query(
    `update task
        set state = 'running', attempts = attempts + 1, claimed_at = now()
      where id in (
        select id from task
         where state = 'ready' and run_at <= now()
         order by run_at
         for update skip locked
         limit $1
      )
    returning id, kind, reservation_id as "reservationId", attempts`,
    [limit],
  );
  return rows;
}
```

Everything else is the failure path. A crashed worker leaves rows stuck in `running`, so a reaper moves anything claimed more than five minutes ago back to `ready`. That reaper is the reason the consumer has to be idempotent: at-least-once is not a design choice you get to decline, it's the consequence of a process that can die between the send and the commit.

```ts
const MAX_ATTEMPTS = 5;

async function run(task: Task): Promise<void> {
  // Dedupe key is stable across attempts: one reminder per reservation, ever.
  const key = `${task.kind}:${task.reservationId}`;
  const claimed = await pool.query(
    `insert into delivery (dedupe_key, task_id) values ($1, $2)
     on conflict (dedupe_key) do nothing returning id`,
    [key, task.id],
  );
  if (claimed.rowCount === 0) {
    await pool.query(`update task set state = 'done' where id = $1`, [task.id]);
    return;
  }

  try {
    await send(task, key); // provider gets the same key as its idempotency header
    await pool.query(`update task set state = 'done' where id = $1`, [task.id]);
  } catch (err) {
    const permanent = (err as { status?: number }).status === 422;
    if (permanent || task.attempts >= MAX_ATTEMPTS) {
      await pool.query(
        `update task set state = 'dead', last_error = $2 where id = $1`,
        [task.id, String(err)],
      );
      return;
    }
    const retryAfter = (err as { retryAfter?: number }).retryAfter // seconds, from a 429
      ?? Math.min(2 ** task.attempts * 15, 900) + Math.floor(Math.random() * 10);
    await pool.query(
      `update task
          set state = 'ready', run_at = now() + ($2 || ' seconds')::interval
        where id = $1`,
      [task.id, retryAfter],
    );
  }
}
```

That's the whole engine. Ship it on a Tuesday.

## Retries, dead letter queues, and what an idempotent consumer really costs

The dead letter state is a column, not a product. What earns its keep is the discipline around it: every dead row keeps the last error and the original payload, and there's one command that replays a filtered set of them after you've fixed the cause. Without replay, a dead letter queue is a landfill with better branding.

Idempotency has a price that nobody quotes up front, and it's not the unique index. It's that the dedupe key has to encode a business rule, and the business rule is often blurrier than the code. One reminder per reservation, ever? Then `hold_reminder:8881` is right. One per reservation per extension, because players who extend a hold expect a second nudge? Then the key needs the extension counter, and every worker that constructs it has to agree — which is where I'd spend the review time, honestly, more than on the queue mechanics.

Three numbers tell you whether any of this is working, and they're all queries against the same table: the age of the oldest task that is due and unclaimed, attempts per successful delivery, and dead rows per hour by kind. Alert on the first one. Queue lag crossing your lateness budget is the only page that means "players are seeing something wrong right now"; CPU graphs on the worker box mean nothing here. Testing follows the same shape — run the consumer twice against the same claimed row in an integration test and assert one delivery row, because that assertion is the entire correctness argument for at-least-once delivery.

## What I'd change at scale, and when I'd stop

The catch is that a polling table is a single write hotspot. Somewhere north of a few thousand tasks a second the `update ... returning` starts contending, and the fixes get progressively less fun: partition by kind, shard `run_at` into buckets, move the claim into a stored procedure to cut round trips. At that point a broker with native delayed delivery is honest value and I'd migrate the delivery path while leaving the hold state where it is.

Stick with a dedicated queue product from day one if you're already running one for other work, if your consumers are in three languages, or if you need sub-second expiry. A database queue also isn't a good fit for long fan-out chains — the moment a task's output is five more tasks with ordering constraints, you want a workflow engine and its retry semantics, not a table and a cron sweep.

I'm not sure the sub-second tier is ever worth it for reservation expiry specifically. Players can't tell the difference between a hold released in 200 ms and one released in 3 seconds, and the second number costs a system I'd have to keep alive. Your mileage may vary if your holds are on matchmaking slots instead of items, where the wait is user-visible and measured in seconds. Sending mail is undifferentiated work worth buying; deciding when a promise expires is your product, so keep it in the database where the invariant already lives.

## Further reading

- MDN: 429 Too Many Requests — https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- MDN: Retry-After header — https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After
- PostgreSQL SELECT documentation, including FOR UPDATE ... SKIP LOCKED — https://www.postgresql.org/docs/current/sql-select.html
- RFC 9110, HTTP Semantics — https://www.rfc-editor.org/rfc/rfc9110.html
