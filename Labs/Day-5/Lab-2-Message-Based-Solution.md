# Lab Guide: Message-Based Solutions with Service Bus and Queue Storage

Oct 16, 2026 · @Nived Varma

In two 40-minute labs you will route NovaTel orders with Azure Service Bus, then process and diagnose support tickets with Azure Queue Storage, using the Azure portal.

## Before you start

**Scenario.** NovaTel sells SIM cards and plans online. Every order must be processed exactly as agreed by Billing, Shipping and Notifications, and no order may be lost. Customers also raise support tickets that a background worker handles. You will build both flows.

| Lab | Service | What you do | Time |
| --- | --- | --- | --- |
| Lab 1 | Azure Service Bus | Create an orders queue, a topic with three subscriptions, a filter, and force a dead-letter | 40 min |
| Lab 2 | Azure Queue Storage | Run a ticket worker, find a stuck poison message, and justify your service choice | 40 min |

**You need.**

- A computer with a web browser and the Azure portal sign-in your trainer gave you.
- Your participant ID and the name of your own resource group, both from your trainer.
- Notepad and Command Prompt, which are already installed on Windows.
- For Lab 2: the folder named `TicketWorker` your trainer provides, containing `TicketWorker.exe`.

**Rules.**

- Work only inside your own resource group.
- Do not delete your resource group. A cleanup script removes everything at the end of the day.
- Wherever a name contains `<id>`, use your participant ID in lowercase, no spaces or symbols. If your ID is `p07`, then `sb-nvt-<id>` becomes `sb-nvt-p07`.

**Your names.** Fill in the last column before you begin.

| Item | Name to use | Your value |
| --- | --- | --- |
| Participant ID | Given by trainer, lowercase |  |
| Resource group | Given by trainer |  |
| Service Bus namespace | `sb-nvt-<id>` |  |
| Queue | `orders` |  |
| Topic | `order-events` |  |
| Subscriptions | `billing`, `shipping`, `notify` |  |
| Storage account (Lab 2) | `stnvt<id>q` (3 to 24 lowercase letters and numbers) |  |
| Storage queues (Lab 2) | `tickets` and `tickets-poison` |  |

**Open the portal.** Go to https://portal.azure.com in your browser and sign in. Close any welcome tour that appears.

**Permission note.** The Service Bus Explorer tool in the portal needs a data role (Service Bus Data Owner) or the access key. Part A tells you how to switch to the access key, so you do not need any extra role.

## Lab 1, Part A: Create the Service Bus namespace (5 min)

A namespace is the container that holds your queues and topics.

1. In the search box at the top of the portal type `Service Bus` and click **Service Bus** under Services.
2. Click **+ Create**.
3. On the **Basics** tab set:
   - **Subscription**: leave the one already selected.
   - **Resource group**: choose **your** resource group. Do not click Create new.
   - **Namespace name**: `sb-nvt-<id>`, for example `sb-nvt-p07`. It must be unique across Azure, 6 to 50 characters, letters, numbers and hyphens only, and must not end in `-sb` or `-mgmt`.
   - **Location**: the same region as your resource group.
   - **Pricing tier**: **Standard**. Topics are not available on Basic, so do not choose Basic.
4. Click **Review + create**, wait for **Validation passed**, then click **Create**.
5. When the page says **Your deployment is complete**, click **Go to resource**.

**Checkpoint:** the namespace Overview page shows **Pricing tier: Standard** and **Status: Active**.

## Lab 1, Part B: Create the orders queue and process one order (10 min)

**Create the queue.**

1. In the left menu, under **Entities**, click **Queues**, then **+ Queue**.
2. In **Name** type `orders`. Leave every other setting at its default. Click **Create**.

**Open Service Bus Explorer and choose how it signs in.**

1. Click `orders` in the list.
2. In the left menu click **Service Bus Explorer**.
3. Click **Settings** in the toolbar. Choose **Access Key** and click **Save**. This avoids needing a special role.

**Send one order.**

1. Click **Send messages**.
2. **Content Type**: **Application/Json**.
3. **Message body**: paste the order below.

```json
{
  "orderId": "9021",
  "subscriberId": "4471",
  "plan": "Plus-50",
  "item": "SIM card"
}
```

4. Open **Custom Properties** and add one property: name `deliveryType`, type **String**, value `physical`. Make sure the value has no spaces.
5. Click **Send**.
6. Look at the **Active Messages** number for the queue. It is now **1**.

**Peek, then receive with a lock.**

1. In the dropdown near the top of the Explorer choose **Peek Mode**, then click **Peek from start**. Select the message in the grid and look at the **Message Body** and **Message Properties** tabs. Peeking does not remove or lock the message.
2. Switch the dropdown to **Receive Mode**, click **Receive messages**, set the receive mode to **PeekLock**, number of messages to **1**, and click **Receive**.
3. The message appears in the grid. It is now **locked** to you. Notice that the Delivery Count in its properties is **1**.
4. Select the message and click **Complete**. This tells Service Bus the work is done, and the message is deleted.
5. Check that **Active Messages** is now **0**.

**Checkpoint:** you sent one order, received it under a lock, and completed it. In one sentence, explain what would have happened if you had clicked **Abandon lock** instead of **Complete**: the message would become available again and its delivery count would go up.

## Lab 1, Part C: Create a topic with three subscriptions (8 min)

A queue gives one message to one worker. A **topic** gives every subscription its own copy, so Billing, Shipping and Notifications can all react to the same order.

**Create the topic.**

1. Go back to your namespace (click `sb-nvt-<id>` in the breadcrumb at the top, or find it in your resource group).
2. In the left menu, under **Entities**, click **Topics**, then **+ Topic**.
3. **Name**: `order-events`. Leave the rest at the defaults. Click **Create**.

**Create the subscriptions.** Click `order-events` to open the topic, then repeat the steps below for each row of the table.

1. Click **+ Subscription**.
2. Type the **Name** from the table.
3. Set **Max delivery count** as shown. The default is 10. Leave every other field at its default.
4. Click **Create**.

| Subscription | Max delivery count |
| --- | --- |
| `billing` | 3 (type 3 instead of 10, you need it in Part F) |
| `shipping` | 10 (leave default) |
| `notify` | 10 (leave default) |

**Checkpoint:** the topic's Subscriptions list shows `billing`, `shipping` and `notify`, each with 0 active messages.

## Lab 1, Part D: Make Shipping receive only physical SIM orders (7 min)

A new subscription accepts every message by default through a rule named `$Default`. You will replace it on `shipping` with a filter.

1. In the topic, click the subscription `shipping`.
2. In the left menu click **Rules** (it may be under **Settings**).
3. Select the rule `$Default` and click **Delete**. Confirm.
4. Click **+ Add rule** (or **Add**).
5. **Name**: `physical-only`.
6. **Filter type**: **SQL Filter** (not Correlation).
7. **SQL filter expression**: type exactly `deliveryType = 'physical'` with single quotes around physical.
8. Click **Save** (or **Add**).

**Checkpoint:** the Rules list for `shipping` now shows only `physical-only`. The names of custom properties are case-sensitive, so `deliveryType` must match what you set when sending.

Leave `billing` and `notify` on their default rule. They accept every order.

## Lab 1, Part E: Publish two orders and compare the counts (5 min)

1. Go to **Topics**, click `order-events`, then click **Service Bus Explorer** in the left menu. If a sign-in choice appears, click **Settings**, choose **Access Key**, and click **Save**.
2. Click **Send messages**. Set **Content Type** to **Application/Json**.
3. **Order 1.** Paste the body below. Add the custom property `deliveryType` (String) with value `physical`. Click **Send**.

```json
{ "orderId": "9022", "subscriberId": "4471", "plan": "Plus-50", "item": "SIM card" }
```

4. **Order 2.** Click **Send messages** again. Paste the body below. Add the custom property `deliveryType` (String) with value `esim`. Click **Send**.

```json
{ "orderId": "9023", "subscriberId": "5120", "plan": "Basic-10", "item": "eSIM" }
```

5. Go back to the topic and open its **Subscriptions** list. Wait a few seconds, click **Refresh**, and fill in the table.

| Subscription | Active messages (expected) | Your count |
| --- | --- | --- |
| `billing` | 2 |  |
| `shipping` | 1 |  |
| `notify` | 2 |  |

**Checkpoint:** your counts match. Write one sentence on why `shipping` has fewer: its rule `physical-only` rejects the eSIM order because an eSIM is not shipped.

**Now prove it.** Open `shipping`, go to its **Service Bus Explorer**, choose **Peek Mode** and **Peek from start**. The one message must be order 9022. Do the same for `billing`: you should see both orders.

## Lab 1, Part F (stretch): Force a message into the dead-letter queue (5 min)

You set `billing` to give up after **3** delivery attempts. You will make a worker "fail" three times on purpose.

1. Open the `billing` subscription, then **Service Bus Explorer**, and choose **Receive Mode**.
2. Click **Receive messages**. Set the mode to **PeekLock**, number of messages **1**, and click **Receive**. Note the **Delivery Count** shown in the message properties: **1**.
3. Select the message and click **Abandon lock**. This pretends your worker crashed. The message is available again.
4. Repeat steps 2 and 3 until the third receive. Delivery Count reaches **3**. Click **Abandon lock** one last time.
5. Look at the metrics at the top of the Explorer. **Active Messages** drops by one and **Dead-lettered Messages** goes up by one.
6. Click the **Dead-letter** tab (the dead-letter sub-queue) and click **Peek from start**.
7. Select the message and open **Message Properties**. Read the dead-letter reason and description, and write them here: \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_. Expected reason: `MaxDeliveryCountExceeded`.

**Checkpoint:** you can explain what the dead-letter queue is, and why it stops one bad message from being retried forever.

**Extra challenge.** Send one more order to the topic, receive it from billing in PeekLock mode, and use **Dead-letter** on a fresh message instead, so the application chooses the reason. Which scenario would justify that, rather than waiting for the delivery count? Example: an order with an invalid plan code, because retrying can never fix it.

## Lab 2, Part G: Create the storage account and two queues (5 min)

Queue Storage lives inside a storage account. It is the simple, low-cost queue.

1. In the portal search box type `Storage accounts` and click **Storage accounts**, then **+ Create**.
2. On the **Basics** tab set:
   - **Resource group**: **your** resource group.
   - **Storage account name**: `stnvt<id>q`, for example `stnvtp07q`. Lowercase letters and numbers only, 3 to 24 characters. If the name is taken, add a digit.
   - **Region**: the same as your resource group.
   - **Performance**: **Standard**. **Redundancy**: **Locally-redundant storage (LRS)**.
3. Click **Review + create**, then **Create**. When the deployment completes click **Go to resource**.
4. In the left menu, under **Data storage**, click **Queues**.
5. Click **+ Queue**, type `tickets`, click **OK**.
6. Click **+ Queue** again, type `tickets-poison`, click **OK**.
7. If the portal shows an authorization message, click **Switch to access key**.

**Checkpoint:** two queues are listed, `tickets` and `tickets-poison`, both empty. Note the idea behind the second name: Queue Storage has **no built-in dead-letter queue**. A *poison queue* is a normal queue that your worker uses by convention to park messages it cannot process.

## Lab 2, Part H: Add six support tickets (5 min)

Five tickets are valid. One, T-1004, is broken on purpose: it is missing its final closing brace `}`. This is your seeded failure.

1. Click the queue `tickets`, then **+ Add message**.
2. Paste **one** ticket from the list below into **Message text**.
3. Set **Expires in** to **7 days**.
4. If you see a checkbox **Encode the message body in base64**, untick it.
5. Click **OK**.
6. Repeat for all six tickets, in order. Copy each one exactly.

```json
{"ticketId":"T-1001","subscriberId":"4471","category":"billing","text":"Charged twice"}
```

```json
{"ticketId":"T-1002","subscriberId":"5120","category":"network","text":"No signal at home"}
```

```json
{"ticketId":"T-1003","subscriberId":"3308","category":"sim","text":"Need a replacement SIM"}
```

```json
{"ticketId":"T-1004","subscriberId":"7781","category":"billing","text":"Wrong plan applied"
```

```json
{"ticketId":"T-1005","subscriberId":"2290","category":"roaming","text":"Roaming charges query"}
```

```json
{"ticketId":"T-1006","subscriberId":"6014","category":"network","text":"Slow data speed"}
```

**Checkpoint:** the `tickets` queue lists six messages. For each, the **Dequeue count** column is **0**, because no worker has read them yet.

## Lab 2, Part I: Run the ticket worker (8 min)

The worker is a small program from your trainer. It reads tickets from `tickets`, processes the valid ones and deletes them. If a ticket keeps failing, it moves it to `tickets-poison` after **5 attempts**. It waits **10 seconds** before a failed message becomes visible again.

**Copy your storage connection string.**

1. In the portal open your storage account `stnvt<id>q`.
2. In the left menu, under **Security + networking**, click **Access keys**.
3. Under **key1**, next to **Connection string**, click **Show**, then click the copy icon.
4. Keep this value on your clipboard. Treat it like a password and never share it.

**Start the worker.**

1. Click the Windows **Start** button, type `cmd`, and press **Enter**. A black Command Prompt window opens.
2. Type ` cd /d  ` (with a space after `d`), then type the folder path your trainer gave you for `TicketWorker`, for example `C:\Labs\TicketWorker`, and press **Enter**.
3. Type ` TicketWorker.exe  ` (with a space after it), then type a double quote `"`, paste the connection string by **right-clicking** in the window, type another double quote `"`, and press **Enter**.
4. Watch the output. You should see lines like these. The exact order can vary.

```text
Worker started. Press Ctrl+C to stop. It stops by itself after 3 minutes.
Processed ticket T-1001
Processed ticket T-1002
Processed ticket T-1003
Processed ticket T-1005
Processed ticket T-1006
FAILED (attempt 1 of 5): could not read ticket, will retry
FAILED (attempt 2 of 5): could not read ticket, will retry
```

Leave this window open. Go to Part J while it runs.

## Lab 2, Part J: Diagnose the stuck ticket (10 min)

You are the on-call engineer. The queue should be empty by now, but one ticket will not go away.

1. In the portal open `stnvt<id>q`, then **Queues**, then `tickets`. Click **Refresh** every 10 seconds.
2. Write down what you see:

| Question | Your answer |
| --- | --- |
| How many messages remain in `tickets`? |  |
| Which ticket is it? (click the message to read it) |  |
| What is its **Dequeue count** at your first look? |  |
| What is its **Dequeue count** 10 seconds later? |  |

3. Read the message text carefully. What is wrong with it? Write the exact problem: \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_. Expected: the closing brace `}` is missing, so the text is not valid JSON.
4. Use this symptom table to name the cause.

| Symptom | Likely cause | First action |
| --- | --- | --- |
| Dequeue count keeps rising on one message | Poison message looping | Move to a poison queue, then fix the payload |
| Many messages waiting, count low | Workers stopped or slow | Check and scale the workers |
| Same work done twice | Visibility timeout shorter than the work | Lengthen it, make the handler idempotent |

Write your diagnosis in one sentence: \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_.

5. Wait until the worker window shows a line that says the ticket was moved to `tickets-poison`. This happens on the **5th** attempt, roughly one minute after the start.
6. Refresh `tickets`: it is now empty. Open `tickets-poison` and refresh: the broken ticket is there.
7. Now think like an operator. What can you **not** tell from the poison queue? Tick all that apply: \[ \] the error text, \[ \] the time it failed, \[ \] how many attempts were made, \[ \] the original queue. Expected answer: unless your worker writes them down, none of these are recorded. A Service Bus dead-letter queue would have given you a reason and description for free.

**Checkpoint:** you found the broken message by its rising Dequeue count, you identified the missing brace, and you confirmed the message was parked in `tickets-poison`. You can stop the worker by pressing **Ctrl+C** in its window, or let it stop after 3 minutes.

## Lab 2, Part K: Choose the right service (6 min)

NovaTel has two new workloads. For each one, decide between **Service Bus** and **Queue Storage**, and write **three sentences** that use these three ideas: **ordering**, **message size** and **delivery guarantees**.

| Workload | Your choice | Your three sentences |
| --- | --- | --- |
| 1. One SIM order has three steps: activate, add the roaming add-on, then send the confirmation. The steps must run in order for each order. |  |  |
| 2. Millions of small survey-feedback records (a few hundred bytes each). Order does not matter and cost matters. |  |  |

Write your answers first. Then compare them with the model answers below.

**Model answer 1: Service Bus.** The three steps need ordering per order, which Service Bus provides with sessions (use the order ID as the session ID). It also gives a built-in dead-letter queue with a reason, so a failed step is easy to find, and a topic can feed downstream teams. Messages are small, so size is not a concern.

**Model answer 2: Queue Storage.** The records are tiny, far below the 64 KB limit, and order is irrelevant. Queue Storage is simple and inexpensive at very large volumes. A poison queue and a dequeue-count guard cover the occasional bad record, and at-least-once delivery is fine if the handler is idempotent.

## Lab 2, Part L: Stretch, add a second worker (5 min)

Finish early? See competing consumers in action.

1. In the portal open `stnvt<id>q`, then **Queues**, then `tickets`. Click **Add message** and add ten new tickets with ids `T-2001` to `T-2010`. Use the same JSON shape as before, with a valid closing `}` each time.
2. Open **two** Command Prompt windows. Start `TicketWorker.exe "<connection string>"` in both, one right after the other.
3. Watch both windows. Each ticket is processed by only one of them.
4. Answer: why does no ticket get processed twice? Expected: while one worker holds a message, it is invisible to the other for the visibility timeout. Once the worker deletes it, it is gone.
5. Answer: what would go wrong if processing took 30 seconds but the visibility timeout were 10 seconds? Expected: the message would reappear while the first worker was still busy, and a second worker could process it too. This is why handlers must be idempotent and the timeout must exceed the real work time.

**Checkpoint:** you can explain which service you would pick for each workload, and you have seen two workers share one queue without duplicating work.

## You are done when

- [ ] Service Bus namespace `sb-nvt-<id>` (Standard) exists with queue `orders` and topic `order-events`.
- [ ] You sent, peeked, received and completed an order on the `orders` queue.
- [ ] Subscriptions `billing`, `shipping` and `notify` exist, and `shipping` uses the rule `deliveryType = 'physical'`.
- [ ] Counts after two orders: billing 2, shipping 1, notify 2, and you can explain why.
- [ ] You know where the Dead-letter tab is and what reason a message gets after too many deliveries.
- [ ] Storage account `stnvt<id>q` has queues `tickets` and `tickets-poison`.
- [ ] You saw ticket T-1004 loop with a rising Dequeue count and end up in `tickets-poison`.
- [ ] You wrote your Service Bus vs Queue Storage choice for both workloads.

Your resource group is deleted tonight by the clean-up script, so there is nothing to delete yourself.

## If something goes wrong

| What you see | Likely cause | What to do |
| --- | --- | --- |
| Service Bus Explorer says you do not have permission | You are using Microsoft Entra sign-in without a data role | In Explorer, open **Settings** and choose **Access Key**, then retry |
| The namespace name is rejected | Names are global and must be unique | Add your participant ID, use only letters, numbers and hyphens |
| No **Topics** option in the menu | The namespace is on the Basic tier | Delete it and recreate it on **Standard** |
| `shipping` receives both orders | The `$Default` rule was not deleted, or `deliveryType` was typed in the body only | Delete `$Default`, and set `deliveryType` as a **custom property** when sending |
| `shipping` receives nothing | Property name or value has different capitalisation | Use exactly `deliveryType` and `physical` (names and values are case-sensitive) |
| Storage name rejected | Needs 3 to 24 lowercase letters and numbers, globally unique | Use `stnvt<id>q` with no hyphens or capitals |
| Added messages look garbled in the worker | Base64 encoding was ticked when adding | Add them again with **Encode the message body in base64** unticked |
| `TicketWorker.exe` is not recognised | Wrong folder in Command Prompt | Use `cd /d` with the folder your trainer gave you, then run it again |
| Worker says authentication failed | Connection string was cut or missing quotes | Copy key1 **Connection string** again and wrap it in double quotes |
| All six tickets processed, T-1004 included | The message was typed with the closing `}` | Add a new ticket with the brace missing and run again |

## Discuss with your group

1. Why did a message stay in the Service Bus queue after you only peeked it, but disappear after you completed it?
2. A shipping rule was set up wrongly and 500 esim orders were delivered to shipping. What would you change, and how would you clean up?
3. Who should own the dead-letter queue and the poison queue in a real NovaTel team? What alert would you set?
4. Your poison queue shows the message but not the reason. What would you add to the worker to make triage easier?
5. Both services can deliver a message twice. Give one NovaTel example where that would hurt, and how an idempotent handler prevents it.

## Trainer appendix: build the ticket worker (not for participants)

Participants only run the finished `TicketWorker.exe`. Build it once on your own machine, test it, then zip it with a one-line instruction for the class. Do not hand out the source unless you want to.

**Before class**

1. Build and run it against a throw-away storage account with the six tickets from Part H. Confirm that T-1001..T-1003, T-1005 and T-1006 are processed and T-1004 ends in `tickets-poison` after about 40 to 60 seconds.
2. Confirm in the Windows sandbox that the .exe starts (antivirus and application-control rules can block unsigned files) and that outbound HTTPS to `*.queue.core.windows.net` is allowed.
3. Confirm participants can reach the share or folder where you put the zip, and tell them the path to use in Part I.

**Build steps (on a machine with the .NET 8 SDK)**

```text
dotnet new console -n TicketWorker
cd TicketWorker
dotnet add package Azure.Storage.Queues
(replace Program.cs with the code below)
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true
```

The finished file is `bin\Release\net8.0\win-x64\publish\TicketWorker.exe`. Zip that single file.

**Program.cs**

```csharp
using System.Text;
using System.Text.Json;
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;

if (args.Length < 1)
{
    Console.WriteLine("Usage: TicketWorker.exe \"<storage connection string>\"");
    return;
}

const int MaxAttempts = 5;                         // move to poison on the 5th failed read
var visibility = TimeSpan.FromSeconds(10);         // short, so the lab is quick
var stopAt = DateTime.UtcNow.AddMinutes(3);        // stops by itself

var tickets = new QueueClient(args[0], "tickets");
var poison = new QueueClient(args[0], "tickets-poison");

using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };

Console.WriteLine("Worker started. Press Ctrl+C to stop. It stops by itself after 3 minutes.");

try
{
    while (!cts.IsCancellationRequested && DateTime.UtcNow < stopAt)
    {
        QueueMessage[] batch = (await tickets.ReceiveMessagesAsync(10, visibility, cts.Token)).Value;

        if (batch.Length == 0)
        {
            await Task.Delay(2000, cts.Token);
            continue;
        }

        foreach (var msg in batch)
        {
            string body = ReadBody(msg.MessageText);
            string? id = TryGetTicketId(body);

            if (id != null)
            {
                Console.WriteLine($"Processed ticket {id}");
                await tickets.DeleteMessageAsync(msg.MessageId, msg.PopReceipt, cts.Token);
            }
            else if (msg.DequeueCount >= MaxAttempts)
            {
                await poison.CreateIfNotExistsAsync(cancellationToken: cts.Token);
                await poison.SendMessageAsync(body, cancellationToken: cts.Token);
                await tickets.DeleteMessageAsync(msg.MessageId, msg.PopReceipt, cts.Token);
                Console.WriteLine($"MOVED to tickets-poison after {msg.DequeueCount} attempts");
            }
            else
            {
                Console.WriteLine($"FAILED (attempt {msg.DequeueCount} of {MaxAttempts}): could not read ticket, will retry");
            }
        }
    }
}
catch (OperationCanceledException) { }

Console.WriteLine("Worker stopped.");

// The portal may or may not Base64-encode the text, so accept both.
static string ReadBody(string text)
{
    try { return Encoding.UTF8.GetString(Convert.FromBase64String(text)); }
    catch (FormatException) { return text; }
}

// A ticket is valid only if it parses as JSON and has a ticketId.
static string? TryGetTicketId(string json)
{
    try
    {
        using var doc = JsonDocument.Parse(json);
        return doc.RootElement.TryGetProperty("ticketId", out var p) ? p.GetString() : null;
    }
    catch (JsonException) { return null; }
}
```

**Notes for the trainer**

- A message left alone becomes visible again after 10 seconds, and each read raises its Dequeue count. That is the behaviour participants watch in Part J.
- A plain-text message that happens to be valid Base64 would be decoded. The sample tickets contain braces and quotes, so they are not valid Base64 and are read as plain text.
- The worker deliberately stores only the body in `tickets-poison`. That is the point of Part J step 7.
- If you want participants to see richer logging, extend the poison branch to write a JSON wrapper with the reason, the time and the attempt count.

## Sources

Microsoft Learn pages consulted on 2026-10-16: Service Bus queues, topics and subscriptions overview; Service Bus dead-letter queues; Service Bus Explorer; Azure Queue Storage introduction; Azure Functions Queue storage trigger (maxDequeueCount and visibilityTimeout settings). Portal menu names and button labels can change, so test every step once before class.
