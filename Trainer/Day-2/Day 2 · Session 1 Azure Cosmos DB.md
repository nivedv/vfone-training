# Day 2 · Session 1: Azure Cosmos DB

Oct 13, 2026 · @Nived Varma

NovaTel Customer Self-Care: trainer demos, participant labs, seeded failures and quiz, all self-contained for Day 2.

## Session at a glance

This session runs about 2 hours: 30 minutes of trainer demos, 90 minutes of participant labs, and 15 minutes for the quiz and debrief. Everything lives in `rg-novatel-day2-<id>`, so nothing depends on Day 1.

| Block | Minutes | Who | Outcome |
| --- | --- | --- | --- |
| Demo 1: CRUD with .NET SDK | 15 | Trainer | Create, read, query, replace, delete customer profiles |
| Demo 2: change feed | 15 | Trainer | Plan upgrade fires a billing handler |
| Lab 1 | 45 | Participant | CRUD plus consistency level choice |
| Lab 2 | 45 | Participant | Change-feed billing notify, fix hot partition |
| Quiz and debrief | 15 | Both | 8 questions |

### Pre-flight setup (run by trainer 60 minutes before, per participant ID)

Free tier allows one account per subscription and is not available on serverless accounts, so each participant gets a **serverless** account. Serverless avoids provisioned RU/s cost and is fine for labs. Account creation takes a few minutes, so create all accounts before the session starts.

```powershell
# setup-s1.ps1  -Id p01  (unique participant id, lowercase letters/digits, max 8 chars)
param([Parameter(Mandatory)][string]$Id, [string]$Location = "centralindia")

$rg      = "rg-novatel-day2-$Id"
$account = "cosmos-novatel-$Id"      # globally unique, 3-44 chars, lowercase
$db      = "novatel"

az group create -n $rg -l $Location --tags course=azure-dev day=2 owner=$Id | Out-Null

az cosmosdb create -n $account -g $rg `
  --capabilities EnableServerless `
  --default-consistency-level Session `
  --locations regionName=$Location failoverPriority=0 isZoneRedundant=False | Out-Null

az cosmosdb sql database create -a $account -g $rg -n $db | Out-Null

# Containers for Demo 1 / Lab 1 (customer profiles)
az cosmosdb sql container create -a $account -g $rg -d $db -n customers --partition-key-path "/customerId" | Out-Null

# Containers for Demo 2 / Lab 2 (change feed needs a leases container)
az cosmosdb sql container create -a $account -g $rg -d $db -n plans --partition-key-path "/customerId" | Out-Null
az cosmosdb sql container create -a $account -g $rg -d $db -n leases --partition-key-path "/id" | Out-Null
az cosmosdb sql container create -a $account -g $rg -d $db -n billing-notifications --partition-key-path "/customerId" | Out-Null

# Seeded hot-partition container: partitioned on /country (3 values) instead of a high-cardinality key
az cosmosdb sql container create -a $account -g $rg -d $db -n usage-events --partition-key-path "/country" | Out-Null

$conn = az cosmosdb keys list -n $account -g $rg --type connection-strings `
  --query "connectionStrings[0].connectionString" -o tsv

# Lab convenience: endpoint + key for the Windows sandbox session
$ep  = az cosmosdb show -n $account -g $rg --query documentEndpoint -o tsv
$key = az cosmosdb keys list -n $account -g $rg --query primaryMasterKey -o tsv
"COSMOS_ENDPOINT=$ep`nCOSMOS_KEY=$key`nPARTICIPANT_ID=$Id" | Out-File "$HOME\novatel-day2-$Id.env" -Encoding ascii
Write-Host "Ready: $account in $rg. Credentials in $HOME\novatel-day2-$Id.env"
```

Seed data: run `.\scripts\setup-s1.ps1 -Id <id> -Seed`, or run `dotnet run --project NovaTel.Seed` after loading the environment. It loads 200 customer profiles (`c-1001` to `c-1200`) into `customers` and 5,000 usage events into `usage-events`, where 90 percent carry `country = "IN"`. That skew is what makes Lab 2's hot partition visible. Demo 1 uses `c-9001` so it never touches seeded data. The complete starter code is in the Session 1 starter pack (`day2-s1.zip`).

**Mixed-level note:** key-based access is used for speed in Day 2. Managed identity and RBAC for Cosmos DB come into play on Day 3, so tell participants this is deliberate.

## Trainer Demo 1: CRUD on NovaTel customer profiles (15 min)

**Story to tell:** NovaTel's Self-Care portal needs a profile store that answers in single-digit milliseconds for a customer opening the app, and scales out when 12 million subscribers check bills on the first of the month.

**Talk track (3 min):** account, database, container, item. A container is the unit of scale, and the partition key decides how data and traffic spread. We chose `/customerId` because each profile read is a point read by customer.

**Live steps (12 min):**

1. Create a console project and add the SDK: `dotnet new console -n NovaTel.Profiles` then `dotnet add package Microsoft.Azure.Cosmos`.
2. Create the client once (singleton) and reuse it. Say why: connection pooling, and creating a client per request is the most common production mistake.
3. Create an item, point-read it, query by plan, replace it (upgrade), then delete it. After each call, print the `RequestCharge` so participants see RU cost: a point read is about 1 RU for a 1 KB item, a cross-partition query costs far more.

```
using Microsoft.Azure.Cosmos;
var endpoint = Environment.GetEnvironmentVariable("COSMOS_ENDPOINT")!;
var key      = Environment.GetEnvironmentVariable("COSMOS_KEY")!;

using var client = new CosmosClient(endpoint, key, new CosmosClientOptions {
    ApplicationName = "NovaTel-SelfCare",
    SerializerOptions = new() { PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase }
});
Container customers = client.GetContainer("novatel", "customers");

// CREATE
var profile = new CustomerProfile("c-9001", "c-9001", "Asha Rao", "IN", "Postpaid-Standard", 49.0m);
var created = await customers.CreateItemAsync(profile, new PartitionKey(profile.CustomerId));
Console.WriteLine($"Created, RU: {created.RequestCharge}");

// READ (point read: cheapest, fastest)
var read = await customers.ReadItemAsync<CustomerProfile>("c-9001", new PartitionKey("c-9001"));
Console.WriteLine($"Read {read.Resource.Name}, RU: {read.RequestCharge}");

// QUERY (parameterised, cross-partition)
var q = new QueryDefinition("SELECT * FROM c WHERE c.plan = @plan").WithParameter("@plan", "Postpaid-Standard");
using var it = customers.GetItemQueryIterator<CustomerProfile>(q);
double queryRu = 0;
while (it.HasMoreResults) { var page = await it.ReadNextAsync(); queryRu += page.RequestCharge; }
Console.WriteLine($"Query RU: {queryRu}");

// UPDATE (replace with optimistic concurrency via ETag)
var upgraded = read.Resource with { Plan = "Postpaid-Premium", MonthlyFee = 79.0m };
await customers.ReplaceItemAsync(upgraded, upgraded.Id, new PartitionKey(upgraded.CustomerId),
    new ItemRequestOptions { IfMatchEtag = read.ETag });

// DELETE
await customers.DeleteItemAsync<CustomerProfile>("c-9001", new PartitionKey("c-9001"));

public record CustomerProfile(string Id, string CustomerId, string Name, string Country, string Plan, decimal MonthlyFee);
```

**Pause points to ask the room:**

- "Why did the point read cost less than the query?" (point read targets one logical partition by id and key; the query fans out.)
- "What happens if two agents upgrade the same customer at once?" (ETag mismatch returns 412; show it by replacing with a stale ETag.)

**Checkpoint:** every participant should see the RU numbers. If someone's client throws 401, the key in their `.env` file is stale: re-run setup.

## Trainer Demo 2: change feed triggers billing on plan upgrade (15 min)

**Story to tell:** when a customer upgrades from Standard to Premium, billing must adjust the next invoice without the portal calling billing directly. The change feed lets billing react to the data change itself, so the portal stays fast and billing stays decoupled.

**Design to draw on the board (2 min):** `plans` container (monitored) to Change Feed Processor (leases container tracks progress) to `BillingHandler` to `billing-notifications` container. The handler is a console worker, so Day 2 does not depend on any Day 1 Function. Mention that the same pattern runs as an Azure Functions Cosmos DB trigger in production.

**Live steps (13 min):**

1. Start the processor in a second terminal and leave it running.
2. In the first terminal, upsert a plan record, then upsert it again with a higher plan.
3. Watch the handler log each change and write a notification document.
4. Stop the processor, make two more changes, restart: the processor resumes from its lease and picks up the missed changes. This is the key teaching moment.

```csharp
// BillingWorker/Program.cs
using Microsoft.Azure.Cosmos;

var client = new CosmosClient(Env("COSMOS_ENDPOINT"), Env("COSMOS_KEY"));
var plans  = client.GetContainer("novatel", "plans");
var leases = client.GetContainer("novatel", "leases");
var notify = client.GetContainer("novatel", "billing-notifications");

ChangeFeedProcessor processor = plans
    .GetChangeFeedProcessorBuilder<PlanChange>("billingProcessor", HandleChangesAsync)
    .WithInstanceName(Environment.MachineName)
    .WithLeaseContainer(leases)
    .WithStartTime(DateTime.UtcNow)           // only new changes
    .Build();

await processor.StartAsync();
Console.WriteLine("Billing processor running. Press Enter to stop.");
Console.ReadLine();
await processor.StopAsync();

async Task HandleChangesAsync(ChangeFeedProcessorContext ctx, IReadOnlyCollection<PlanChange> changes, CancellationToken ct)
{
    foreach (var c in changes)
    {
        Console.WriteLine($"[{ctx.LeaseToken}] {c.CustomerId}: {c.Plan} @ {c.MonthlyFee}");
        var note = new BillingNotification(Guid.NewGuid().ToString(), c.CustomerId,
            $"Plan changed to {c.Plan}; prorate from {DateTime.UtcNow:yyyy-MM-dd}", DateTime.UtcNow);
        await notify.CreateItemAsync(note, new PartitionKey(note.CustomerId), cancellationToken: ct);
    }
}

static string Env(string k) => Environment.GetEnvironmentVariable(k) ?? throw new InvalidOperationException(k);

public record PlanChange(string Id, string CustomerId, string Plan, decimal MonthlyFee);
public record BillingNotification(string Id, string CustomerId, string Message, DateTime CreatedUtc);
```

```csharp
// Producer snippet (run in terminal 1): simulate a plan upgrade
await plans.UpsertItemAsync(new PlanChange("c-2001", "c-2001", "Postpaid-Premium", 79.0m), new PartitionKey("c-2001"));
```

**Teach explicitly:**

- The change feed is ordered **per partition key**, not across the whole container, and it records inserts and updates but not hard deletes. Use a soft-delete flag plus TTL if downstream needs delete signals.
- The processor gives at-least-once delivery, so the handler must be idempotent. Ask: "What stops us double-billing if a batch is replayed?" (answer: derive the notification id from the change, for example `customerId + _lsn`, instead of a random Guid; show this as the production fix.)
- Run two instances to show lease distribution (optional, 2 min).

**Checkpoint:** participants who watched the replay after restart can explain why `leases` exists.

## Participant Lab 1: customer profile CRUD and consistency choice (45 min)

**Scenario.** You are on NovaTel's Self-Care team. The portal must let customers view and change their plan. Build the data access layer, then recommend a consistency level for three real read/write patterns.

**You start with:** a starter solution `NovaTel.Lab1` (Program.cs with TODO markers, `CustomerProfile` record, 200 pre-seeded profiles in `customers`), and your own `COSMOS_ENDPOINT` / `COSMOS_KEY` in `%USERPROFILE%\novatel-day2-<id>.env`. Use your own account only: `cosmos-novatel-<id>`.

### Part A: CRUD (25 min)

1. Load your environment variables from the `.env` file and build a singleton `CosmosClient`.
2. **Create:** insert a new profile with id `c-<yourId>-01` using `/customerId` as the partition key.
3. **Read:** point-read the same profile and print the RU charge.
4. **Query:** list all `Postpaid-Standard` profiles for `country = "IN"` using a parameterised query. Print total RU.
5. **Update:** upgrade the profile's plan using `ReplaceItemAsync` with `IfMatchEtag`. Then deliberately replace with a stale ETag and capture the 412 `CosmosException`.
6. **Delete:** remove your test profile.
7. **Record:** fill in this table in your notes.

| Operation | RU charge | Latency (ms) | Observation |
| --- | --- | --- | --- |
| Create |  |  |  |
| Point read |  |  |  |
| Query (filtered) |  |  |  |
| Replace |  |  |  |

**Success check:** your point read costs roughly 1 RU and is cheaper than the query; you have caught a 412 exception.

### Part B: pick the consistency level (20 min)

Cosmos DB offers five levels: Strong, Bounded staleness, Session, Consistent prefix and Eventual. Your account default is **Session**. A client can relax consistency per request but cannot make it stronger than the account default.

In code, read the account default (`client.ReadAccountAsync()`), then override per request:

```csharp
var opts = new ItemRequestOptions { ConsistencyLevel = ConsistencyLevel.Eventual };
var r = await customers.ReadItemAsync<CustomerProfile>("c-1001", new PartitionKey("c-1001"), opts);
```

For each scenario, choose a level and justify it in one sentence covering latency, cost and correctness. Your trainer will review these in the debrief.

| Scenario | Your level | Justification |
| --- | --- | --- |
| 1. A customer upgrades their plan in the app and immediately sees their new plan on the next screen (same user, same session) |  |  |
| 2. A call-centre agent confirms a customer's remaining prepaid balance before authorising a top-up refund |  |  |
| 3. A network operations wall-board shows approximate live usage counts per region, refreshed every few seconds |  |  |

**Hint:** ask three questions. Must the reader see their own write? Can a stale read cause financial harm? Is slightly old data acceptable if it is cheaper and faster?

### Stretch task (for fast finishers)

- Set a **TTL** on the container (`DefaultTimeToLive = -1`) and give a temporary `otp-session` item a 60-second `ttl`. Confirm it disappears.
- Add an **indexing policy** that excludes `/notes/*` and verify with the query metrics that RU on writes drops.
- Write a one-paragraph note: what would change in your consistency recommendation if NovaTel enabled multi-region writes across two countries?

### Reference answers (trainer only)

1. **Session.** Read-your-writes within a session at low latency, cheapest sensible choice for a portal.
2. **Strong** or **Bounded staleness**. A stale balance can cause a wrong refund. Strong costs latency and is limited when regions are far apart, so bounded staleness with a tight window is the practical enterprise answer.
3. **Eventual** (or Consistent prefix). Lowest latency and RU cost, and slightly old counts are acceptable.

## Participant Lab 2: billing notification via change feed, and fixing a hot partition (45 min)

**Scenario.** Two incidents landed on your desk. (1) Billing wants to be told whenever a customer changes plan, without the portal calling billing. (2) Platform monitoring says the usage-events container is skewed towards one country and will not survive the month-end spike.

**You start with:** starter solution `NovaTel.Lab2` containing `BillingWorker` (processor wired, handler body marked TODO) and `UsageLoader` (writes events). Containers `plans`, `leases`, `billing-notifications` and the seeded `usage-events` (partition key `/country`) already exist in your account.

### Part A: change-feed billing notification (20 min)

1. Open `BillingWorker/Program.cs`. Complete `HandleChangesAsync` so that for each changed plan it writes a `BillingNotification` to `billing-notifications`.
2. Make the handler **idempotent**: set the notification `Id` to `$"{c.CustomerId}-{c.Plan}-{c.MonthlyFee}"` and use `UpsertItemAsync` instead of `CreateItemAsync`, so a replayed batch cannot create duplicates.
3. Start the worker. In a second terminal, run `dotnet run --project PlanSimulator -- upgrade c-3001 Postpaid-Premium 79`.
4. Confirm one notification appears in the portal Data Explorer under `billing-notifications`.
5. Stop the worker, run two more upgrades, restart the worker, and confirm both are delivered.

**Success check:** upgrade, stop, restart still yields exactly one notification per upgrade.

### Part B: fix the hot partition (25 min)

The seeded container `usage-events` uses `/country` as its partition key. About 90 percent of the events are for `IN`. A logical partition is capped at 20 GB of data and 10,000 RU/s, so a key with three values caps the whole container well below what a high-volume telecom needs.

1. **Measure the skew.** In Data Explorer, run:

```sql
SELECT c.country, COUNT(1) AS events FROM c GROUP BY c.country
```

2. **Write down the access pattern.** The dominant queries are (a) usage for one customer in a billing month, and (b) all events written per second at ingestion time. Which property is high-cardinality and appears in query (a)?
3. **Compare three candidate keys** and fill in the table.

| Candidate key | Cardinality | Even write spread? | Supports query (a) in one partition? | Verdict |
| --- | --- | --- | --- | --- |
| `/country` |  |  |  |  |
| `/customerId` |  |  |  |  |
| `/country` + `/customerId` (hierarchical) |  |  |  |  |

4. **Create the fixed container.** A partition key cannot be changed on an existing container, so create a new one and copy the data:

```powershell
az cosmosdb sql container create -a cosmos-novatel-<id> -g rg-novatel-day2-<id> -d novatel `
  -n usage-events-v2 --partition-key-path "/customerId"
```

5. **Migrate.** Complete the TODO in `UsageLoader/Migrate.cs`: read pages from `usage-events` with a query iterator and write them to `usage-events-v2` using `CreateItemAsync` with `new PartitionKey(item.CustomerId)`. Enable bulk mode with `AllowBulkExecution = true` on the client options to see the throughput gain.
6. **Re-measure.** Run the skew query against `usage-events-v2` grouped by `customerId`. Then run the same "usage for one customer" query on both containers and compare RU charge.

**Success check:** the per-customer query on `usage-events-v2` runs against a single partition and costs less than on `usage-events`.

### Stretch tasks

- Create `usage-events-v3` with **hierarchical partition keys** (`/country`, `/customerId`) and discuss when it beats a single key.
- Convert the worker to an **Azure Functions Cosmos DB trigger** (isolated worker model) and explain what the platform now manages for you.
- Add `WithMaxItems(10)` and `WithPollInterval(TimeSpan.FromSeconds(1))` to the processor and measure the effect on latency.

### Reference answers (trainer only)

- `/country`: cardinality 3, uneven, scatters query (a): **reject**.
- `/customerId`: millions of values, even writes, query (a) hits one partition: **choose**.
- Hierarchical: best of both for multi-tenant skew, requires newer SDK and adds modelling complexity: **good stretch discussion**.
- Idempotent Id means a replayed batch overwrites, not duplicates.

## Quiz: Cosmos DB (8 questions)

Use as a live poll or handout. Answers are in the trainer column.

| # | Question | Options | Answer |
| --- | --- | --- | --- |
| 1 | Which call is the cheapest way to fetch one customer profile? | A. Cross-partition query. B. Point read with id and partition key. C. Query with `SELECT *`. D. Change feed. | B |
| 2 | A request returns HTTP 412 on a replace. What happened? | A. Throttled. B. Partition full. C. ETag mismatch, someone updated the item first. D. Wrong key. | C |
| 3 | A portal user must see their own write right after saving, at low cost. Which level? | A. Strong. B. Session. C. Eventual. D. Bounded staleness. | B |
| 4 | Can a client request stronger consistency than the account default on a single read? | A. Yes. B. No, only equal or weaker. | B |
| 5 | What does the leases container do for the change feed processor? | A. Stores billing data. B. Tracks progress and shares work across instances. C. Holds indexes. D. Enforces TTL. | B |
| 6 | Change feed delivery is at-least-once. What must your handler be? | A. Fast. B. Single-threaded. C. Idempotent. D. Transactional. | C |
| 7 | Which partition key is the best fit for 5,000 usage events per second across millions of customers? | A. `/country`. B. `/planType`. C. `/customerId`. D. `/month`. | C |
| 8 | Can you change the partition key of an existing container in place? | A. Yes, in the portal. B. No, create a new container and copy data. | B |

**Debrief prompt (5 min):** "Where in NovaTel's real platform would you choose Session, and where would a mistake cost money?" Let the group argue it out using their Lab 1 table.

## Trainer notes

**Timing checkpoints:** by minute 10 of Lab 1 everyone has a working client; by minute 20 of Lab 2 the worker should be running; if more than a third of the room is behind at the Part B start, run the migration as a live demo and let participants follow.

**Common failures and quick fixes:**

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| 401 Unauthorized | Stale or wrong key in `.env` | Re-run `setup-s1.ps1` for that ID, reload env vars |
| 409 Conflict on create | Item id already exists | Use a new id or `UpsertItemAsync` |
| 404 on point read | Wrong partition key value | Key must equal the item's `customerId` |
| 429 Too Many Requests | Serverless burst limit or bulk load too aggressive | Lower concurrency, honour `RetryAfter` |
| Worker sees no changes | `WithStartTime(UtcNow)` set after the upgrade ran | Run the simulator after the worker starts |
| Duplicate notifications | Handler uses random Guid | Use deterministic id plus upsert |

**Honest limits to tell the room:**

- With small seeded data, the account has a single physical partition, so the portal's partition-level hot-partition metrics will not light up. The lab proves the problem through the key's cardinality, the group-by skew and the RU comparison, which is how you would reason before production traffic arrives. Use the Insights blade screenshots if you want to show the physical-partition view.
- Serverless is chosen for cost and for the one-free-tier-per-subscription limit. Production NovaTel billing would use provisioned autoscale.
- Key-based access is a Day 2 shortcut. Day 3 replaces it with managed identity and Entra ID.

**Cleanup:** `az group delete -n rg-novatel-day2-<id> --yes --no-wait` at the end of the day (the combined Day 2 cleanup script will cover all three sessions).

**Sources:** [Cosmos DB free tier limits](https://learn.microsoft.com/azure/cosmos-db/free-tier) · [Cosmos DB quotas and limits](https://learn.microsoft.com/azure/cosmos-db/concepts-limits) · [Change feed processor](https://learn.microsoft.com/azure/cosmos-db/change-feed-processor). Documentation pages were checked on 2026-10-13; SDK code should be compiled and run once against a test account before delivery.

## What participants should learn (teaching storyline)

The code is only the vehicle. By the end of Session 1 a participant should be able to **design a Cosmos DB container for a real workload and defend the choices**, not just call the SDK. Teach one story, NovaTel's customer self-care app, and let every demo answer a question the previous one raised.

### Learning outcomes

By the end, participants can:

1. Choose a partition key for a given workload and say why alternatives fail.
2. Predict whether an operation will be cheap or expensive in RUs, and explain why.
3. Pick a consistency level for a given business scenario.
4. Protect concurrent updates with ETags instead of last-write-wins.
5. Build event-driven processing from the change feed, and explain why the handler must be idempotent.
6. Recognise a hot partition and fix it by redesigning the key.

### The storyline (about 90 min of teaching, then labs)

| Step | The question NovaTel asks | Concept you teach | Proof participants see |
| --- | --- | --- | --- |
| 1 | Where does a customer profile live, and how do we fetch it in milliseconds at 10 million customers? | Account, database, container, item. Logical vs physical partitions. The partition key is the address. | Demo 1: point read of one item |
| 2 | Why is looking up by id so cheap but a search by plan so costly? | Request Units as the currency. Point read vs single-partition query vs cross-partition fan-out. | Demo 1: RU of read vs query side by side |
| 3 | Two agents edit the same customer at once. Who wins? | Optimistic concurrency with ETag; HTTP 412 is a feature. | Demo 1: stale-ETag replace fails |
| 4 | Customer upgrades a plan in the app; how does billing hear about it without polling? | Change feed as a built-in event log. Leases, ordering per partition key, at-least-once delivery. | Demo 2: upgrade triggers a notification |
| 5 | The worker crashes for an hour. Do we lose upgrades? Can we double-bill? | Checkpointing in leases; idempotent handlers. | Demo 2: stop, upgrade twice, restart |
| 6 | Usage data is growing and one key is hot. What went wrong? | Partition key choice: cardinality, even spread, alignment with queries. A key cannot be changed in place; migrate to a new container. | Lab 2 Part B: 90% of events in one key, then the redesign |

### How to run every demo: Predict, Show, Reveal, Name

Typing code live teaches typing. Before each run, stop and make participants commit to an answer:

1. **Predict.** Ask the room, "What will this cost? Will it succeed?" Hands up or chat vote.
2. **Show.** Run it.
3. **Reveal.** Compare with the predictions. The gap is the learning moment.
4. **Name.** Only now give the concept its name and a one-line rule.

### Demo 1 talking points (CRUD)

- **Predict before the point read and query:** "Which is cheaper, and by how much?" Expect guesses of about the same. The point read is about 1 RU for a 1 KB item; the cross-partition query is several times more. **Rule: id plus partition key is the fast path, so design the key around how the app reads.**
- **Predict before the stale-ETag replace:** "Two writers, same item, what happens to the second?" **Rule: `_etag` plus `IfMatchEtag` turns lost updates into a visible 412 that you retry, not silent overwrites.**
- **Show the singleton client.** One `CosmosClient` per app. **Rule: creating one per request exhausts connections; this is the most common production bug.**
- **Whiteboard (2 min, no code):** draw 3 physical partitions, hash the key, show which one serves the read. Point out that a query with no key must visit all three.

### Demo 2 talking points (change feed)

- **Predict:** "If I stop the worker and upgrade two customers, then restart, what does the worker see?" Most say nothing. Reveal the lease holding its place. **Rule: the change feed is a durable log, and the lease is the bookmark.**
- **Predict:** "Can the same change be delivered twice?" Yes, after a crash before the checkpoint. **Rule: at-least-once means the handler must be idempotent. Show the deterministic id and the upsert, and ask what would happen with a random GUID and a plain create.**
- **Say it plainly:** order is guaranteed per partition key, not across the whole container.
- **Production bridge:** the same pattern is the Azure Functions Cosmos DB trigger, which Day 3 builds on.

### Concepts to cover on slides (maximum 4 slides, 10 min)

1. **Consistency spectrum:** strong, bounded staleness, session, consistent prefix, eventual. Use the telecom framing: balance display and fraud check need a stronger read; a promotional banner can be eventual. Session is the default because a user always reads their own writes.
2. **Throughput models:** provisioned, autoscale, serverless. This course uses serverless for cost, but explain when each is right (steady heavy traffic favours provisioned or autoscale).
3. **Partition key checklist:** high cardinality, even distribution, appears in your most frequent queries, never a value that grows without bound in a single key, 20 GB limit per logical partition.
4. **Modelling:** embed what you read together, reference what grows unbounded or changes independently. This is the shift from relational thinking.

### Lab design: what each lab is really teaching

- **Lab 1 teaches cost-awareness.** After finishing the CRUD calls, participants must record the RU of each operation and answer, "Which one would you remove from the hot path, and why?"
- **Lab 2 Part A teaches reliability.** They must explain why a non-idempotent handler would create duplicate notifications after a restart, then make theirs idempotent.
- **Lab 2 Part B teaches design.** They measure the skew themselves, explain why "country" fails as a key, choose a better one, migrate, and re-measure. The before-and-after RU numbers are the proof.
- **Stretch tasks** (for experienced participants) go deeper on hierarchical partition keys, indexing policy, and bulk mode.

### Check for understanding (end of session, 5 min)

Use the quiz in this doc, plus three spoken scenario questions:

1. "A new dashboard queries all customers on a given plan every minute. What do you change?"
2. "Billing double-charged a customer after a worker restart. Where do you look first?"
3. "Your colleague proposes `/plan` as the partition key. Argue against it."

### Common misconceptions to listen for

- "Cosmos DB is just MongoDB or SQL in the cloud." The cost model and partition key are what differ.
- "More RU/s fixes a hot partition." It does not; throughput is divided across partitions.
- "The change feed shows deletes." Not by default; use a soft-delete flag.
- "Strong consistency is always safer." It costs latency and availability, so pick by business need.

## Trainer runbook: setup, test, demo

Work through this in order. Phases 0 to 2 (about 50 minutes) get a working, tested environment; phases 3 and 4 are the two demos exactly as you deliver them; phase 5 cleans up. Run phases 0 to 4 once the day before as a rehearsal, then on the day repeat only the Phase 1 checks and phases 3 and 4.

| Setting | Value |
| --- | --- |
| Id | `nvtrain` |
| Resource group | `rg-nvtl-oct26-trainer` (already exists; setup reuses it and never deletes it) |
| Owner tag | `novatel-trainer` |
| Cosmos account created | `cosmos-novatel-nvtrain` |
| Working folder | `C:\novatel\day2-demo` |

### Phase 0: Prerequisites (10 min, once)

1. Install the Azure CLI and the .NET 8 SDK. Check with `az --version` and `dotnet --version`.
2. Sign in and select the subscription: `az login`, then `az account set --subscription <subscription-id>`.
3. Register the Cosmos provider once: `az provider register -n Microsoft.DocumentDB`. Confirm with `az provider show -n Microsoft.DocumentDB --query registrationState -o tsv` (expect `Registered`).
4. Unzip `day2-s1.zip` to `C:\novatel\day2-s1`. This stays pristine for participants. Make a separate copy for your demos, because the solutions overwrite the starter files: `Copy-Item C:\novatel\day2-s1 C:\novatel\day2-demo -Recurse`.
5. Prove NuGet works: `cd C:\novatel\day2-demo` then `dotnet restore NovaTel.Seed`. Expect a clean restore. If it fails, your network blocks NuGet; fix this before anything else.

### Phase 1: Create the environment (10 min)

```powershell
cd C:\novatel\day2-demo
.\scripts\setup-s1.ps1 -Id nvtrain -ResourceGroup rg-nvtl-oct26-trainer -Owner novatel-trainer -Seed
. .\scripts\load-env.ps1 -Id nvtrain
```

The account takes a few minutes to create. Pass checks:

1. The output says `Reusing existing resource group rg-nvtl-oct26-trainer`.
2. In the portal, account `cosmos-novatel-nvtrain` exists in that group with five containers: `customers`, `plans`, `leases`, `billing-notifications`, `usage-events`.
3. In Data Explorer, `customers` holds 200 items. Run `SELECT VALUE COUNT(1) FROM c` on `usage-events` and expect 5000.

### Phase 2: Smoke test with the solutions (30 min)

The goal is to prove every lab works and to capture the real RU and timing numbers you will quote in class.

| # | Do this | You should see |
| --- | --- | --- |
| 1 | `Copy-Item .\trainer-solutions\Lab1\Operations.cs .\NovaTel.Lab1\Operations.cs -Force` then `dotnet run --project NovaTel.Lab1` | Create and point read about 1 to 10 RU, query costing more than the point read, a 412 for the stale ETag, account default `Session` |
| 2 | `Copy-Item .\trainer-solutions\Lab2\BillingWorker\Program.cs .\NovaTel.Lab2\BillingWorker\Program.cs -Force`. Terminal 1: `dotnet run --project NovaTel.Lab2\BillingWorker`. Wait 10 seconds. Terminal 2: `dotnet run --project NovaTel.Lab2\PlanSimulator -- upgrade c-3001 Postpaid-Premium 79` | Worker logs the change; exactly one document in `billing-notifications` |
| 3 | Stop the worker (Enter), run two more upgrades (`c-3002`, `c-3003`), restart the worker | Both are delivered after the restart, with no duplicates |
| 4 | `dotnet run --project NovaTel.Lab2\UsageLoader -- skew usage-events` | 3 distinct values, `IN` about 90% |
| 5 | `... UsageLoader -- customer-usage usage-events c-1001` | Reported as cross-partition, with an RU number |
| 6 | `az cosmosdb sql container create -a cosmos-novatel-nvtrain -g rg-nvtl-oct26-trainer -d novatel -n usage-events-v2 --partition-key-path "/customerId"` | Container created |
| 7 | `Copy-Item .\trainer-solutions\Lab2\UsageLoader\Migrate.cs .\NovaTel.Lab2\UsageLoader\Migrate.cs -Force` then `... UsageLoader -- migrate usage-events usage-events-v2` | Migrated 5000 items, with elapsed time and RU |
| 8 | `... UsageLoader -- skew usage-events-v2` then `... -- customer-usage usage-events-v2 c-1001` | Many distinct keys; query reported as single partition, with lower RU than step 5 |

Write down the numbers from steps 1, 5, 7 and 8. Check that the whole lab fits its 45-minute slots at participant speed.

### Phase 3: Demo 1, CRUD with the .NET SDK (15 min)

**Before you start:** portal open on Data Explorer for `customers`; PowerShell open in `C:\novatel\day2-demo` with the environment loaded; terminal font enlarged.

1. **Orient (2 min).** In Data Explorer, open one seeded item. Point out `id`, `customerId`, and the system properties `_etag` and `_ts`. Say: the partition key is `/customerId`.
2. **Draw the model (3 min).** Account, database, container, item. Explain that the container is the unit of scale and the partition key decides how data and traffic spread.
3. **Create the project live (2 min).**

```powershell
mkdir C:\novatel\demo1; cd C:\novatel\demo1
dotnet new console -n NovaTel.Profiles; cd NovaTel.Profiles
dotnet add package Microsoft.Azure.Cosmos
```

4. **Code and run (8 min).** Paste the Demo 1 code from the Trainer Demo 1 section above (item id `c-9001`, so seeded data is never touched), then `dotnet run`. After the create, add `Console.ReadLine();` so you can switch to Data Explorer, refresh, and show the new item before deleting it.
5. **Hit the two pause points.** Ask why the point read cost less than the query. Then replace once more with the old ETag to show HTTP 412.

**Pass:** every operation prints its RU, the item appears and disappears in Data Explorer, and the 412 shows.

### Phase 4: Demo 2, change feed triggers billing (15 min)

**Before you start:** the solution `BillingWorker` from Phase 2 is in place. Data Explorer open on `billing-notifications`, with `leases` in another tab. Two terminals, both in `C:\novatel\day2-demo` with the environment loaded.

1. **Draw the flow (2 min).** `plans` to change feed processor (leases track progress) to billing handler to `billing-notifications`. Say that the same pattern runs as an Azure Functions Cosmos DB trigger in production.
2. **Terminal 1:** `dotnet run --project NovaTel.Lab2\BillingWorker`. Wait for `Billing processor running`, then 10 more seconds.
3. **Terminal 2:** `dotnet run --project NovaTel.Lab2\PlanSimulator -- upgrade c-2001 Postpaid-Premium 79`. The worker prints the change within seconds; refresh `billing-notifications` to show the new document.
4. **Replay moment.** Press Enter in terminal 1 to stop the worker. In terminal 2, upgrade `c-2002` and `c-2003`. Restart the worker: both changes arrive. Open `leases` and show the stored progress. This is the key teaching moment.
5. **Teach the two rules.** The feed is ordered per partition key, not across the container. Delivery is at-least-once, so the handler is idempotent: show the deterministic notification id and `UpsertItemAsync` in the code.
6. **Optional (2 min).** Start a second worker in a third terminal and show the leases splitting between instances.

**Pass:** one notification per upgrade, and the replay after restart delivers everything missed.

### Phase 5: Reset and cleanup

- **Re-run a demo:** no reset needed. Use new customer ids (`c-2004`, `c-2005`) for Demo 2; Demo 1 deletes its own item.
- **Clean up** (keeps `rg-nvtl-oct26-trainer`, removes only the lab resources). First preview, then run for real:

```powershell
.\scripts\cleanup-day2.ps1 -Id nvtrain -ResourceGroup rg-nvtl-oct26-trainer -Owner novatel-trainer -Preview
.\scripts\cleanup-day2.ps1 -Id nvtrain -ResourceGroup rg-nvtl-oct26-trainer -Owner novatel-trainer
```

Pass: the preview lists only `cosmos-novatel-nvtrain`; afterwards the resource group still exists and the account is gone.

### If something fails during a demo

| Symptom | Fix | Fallback |
| --- | --- | --- |
| 401 Unauthorized | Re-run `. .\scripts\load-env.ps1 -Id nvtrain` in that terminal | Run the same call from Data Explorer |
| Worker prints nothing after an upgrade | You upgraded before the worker finished starting; upgrade again | Show the notification created in the earlier rehearsal |
| `dotnet add package` hangs | NuGet blocked or slow | Switch to the prebuilt `NovaTel.Lab1` solution and walk through it |
| Cosmos name already taken | Choose a different Id (letters and digits, up to 8) | None needed |
