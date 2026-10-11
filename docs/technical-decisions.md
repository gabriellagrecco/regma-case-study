# Technical decisions and trade-offs

🇺🇸 English · [🇧🇷 Português](decisoes-tecnicas.md)

---

## Postgres with Prisma, not just the Supabase client

With 59 models and dense relations, writing loose SQL would have become a source of silent
errors. Prisma gives a versioned schema, migrations and end-to-end typing: the database and
TypeScript stop being able to disagree.

Supabase stays as database and storage infrastructure. The choice was not between the two, it
was using each one for what it solves best.

**Trade-off:** one more layer and a migration step on every schema change. In exchange,
refactoring across 59 models stops being guesswork.

---

## The agency's client is not a platform user

They should not see the agency dashboard, should not hold permissions, should not show up in a
seat count. They need a narrow door: view, approve, comment.

A separate session makes that boundary explicit in the architecture, instead of depending on a
role check repeated on every screen, which is exactly where this kind of rule tends to leak.

**Trade-off:** two authentication systems to maintain. The gain is that a bug in the dashboard
does not open the portal, and the other way around.

---

## Approval over WhatsApp, not over email

This is the product decision the project came from. An agency's client does not open email and
does not learn a new tool; they open WhatsApp. A notification with a direct link removes exactly
the step where the process used to die.

**Trade-off:** depending on a channel that was not built for this, and the complexity of keeping
a WhatsApp integration alive. It was taken on because it is the product's differentiator.
Without it, Regma would be one more dashboard the client never opens.

---

## A WhatsApp server of our own, decided by margin

A persistent WhatsApp session does not survive in serverless: the function spins up, answers and
dies, and the connection dies with it. So WhatsApp always had to live outside the application.
The question was where.

**First version:** a custom engine on Baileys, running locally. It went down whenever the
machine shut off.

**The obvious alternative was a paid provider**, and it was ruled out on the numbers: Z-API and
similar charge **per number** (~R$100/month each). On a platform whose job is to supply the
connection for every customer, per-unit cost destroys the margin as the base grows. A server of
our own holds dozens of numbers at a fixed cost.

**Decision:** a self-hosted Evolution server, one instance per account, webhook pointing back at
the application.

**Trade-off:** infrastructure of our own to monitor, in exchange for a cost that does not scale
with the base. Unit cost determined the architecture, not the other way around.

---

## A wide scope in a first version

Regma covers CRM, editorial, WhatsApp, email and finance. That is a lot for a v1, and it is a
deliberate choice: **the pain the product came from is precisely the fragmentation between
tools.** A product that solved only one part would hand the user back to the original problem.

**Trade-off accepted:** more surface to maintain and more time until launch, in exchange for
solving the whole pain instead of a slice of it.

---

## What I learned

**How I found out: by using it.** I tested after every build, but the real problems did not show
up in testing. They showed up when I started running a real client inside a tool that was still
in development. None of the three below would have appeared any other way.

### In-memory rate limiting does not exist in serverless

**What I expected:** keeping login attempts in an in-memory `Map` would be enough to block brute
force.

**What happened:** it did not block. On Vercel each instance has its own `Map`, the state
disappears on a cold start and is not shared between instances, so the counter never reached the
limit.

**And there was a second problem, a worse one:** the key was the slug, not the IP. That would
let an attacker get around the limit by changing the identifier, and, in the other direction,
lock a legitimate client out.

**What I changed:** persisted counting, keyed by IP plus identifier.

**What stays with me:** in serverless, *no* state survives in memory between requests. It looks
obvious afterwards; it is not obvious before.

### A Prisma migration does not go through the Supabase pooler

**What I expected:** to use the same connection URL for runtime and for migrations.

**What happened:** the migration hung. Prisma's schema engine does not work through the pooler
(port 6543, pgbouncer), it needs a direct connection.

**What I changed:** `DIRECT_URL` on port 5432 for migrations; runtime keeps going through the
pooler, with the `@prisma/adapter-pg` driver adapter.

**What stays with me:** an application connection and a migration connection have different
requirements. The pooler optimizes one and breaks the other.

### A stateful process needs a machine that does not sleep

**What I expected:** running the WhatsApp engine (Baileys) locally would be enough for the
testing phase.

**What happened:** every time my computer shut off, the connection dropped. And it was not a
bug, it was the nature of the thing: a persistent session requires a host that is always up.

**What I weighed:** hiring a paid provider would solve stability, but it charges per number
(~R$100/month each), which breaks the margin of a platform that has to supply the connection for
all of its customers.

**What I changed:** a self-hosted Evolution server, fixed cost, one instance per account.

**What stays with me:** "where is this going to run" is an architectural decision, not a deploy
detail. And a vendor's pricing model can be incompatible with your product's business model, so
it is worth doing the math before picking the tool.

### The first fix was not the cause

**The symptom:** the application was dropping database connections intermittently. Nothing
obvious in the logs, and the error did not happen every time.

**First hypothesis:** a connection limit on the database URL. I adjusted `connection_limit` in
`DATABASE_URL` and shipped. It looked solved.

**It was not.** The drops came back. The real cause was another layer: the `pg` driver adapter's
pool, which was opening more connections than the database could take, regardless of what the
URL said. I only closed it the next day.

**What stays with me:** the dangerous bug is not the one you cannot solve, it is the one you
think you solved. The first fix lowered the symptom enough for me to believe it was over, and
that is what cost the time. Today I distrust a fix that works when I cannot explain exactly
*why* it used to work and does not now.

> On the record in the repository: attempt on 21 July at 13:23 (`connection_limit` in the URL);
> real cause identified on 22 July at 20:30 (`pg adapter pool, the real cause of the drops`).

---
