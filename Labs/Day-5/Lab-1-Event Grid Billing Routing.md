# Lab 1: Route a NovaTel Billing Event with Azure Event Grid

Oct 16, 2026 · @Nived Varma

In 40 minutes you will make Azure Event Grid send a notification to a handler every time a new NovaTel invoice file arrives, using only the Azure portal and no code.

## Before you start

**Scenario.** NovaTel Finance saves each new invoice as a file in cloud storage. The Billing system must be told the moment a file arrives, without anyone polling the storage. You will build that notification path with Azure Event Grid.

**What you will build.** Four pieces, each one a concept from the session:

| Event Grid concept | What you create in this lab |
| --- | --- |
| Source (publisher) | A storage account with a container named `invoices` |
| Topic | A system topic that Azure creates for you when you subscribe |
| Event subscription | A rule named `evs-billing-created` that sends only Blob Created events |
| Handler | A storage queue named `billing-events` that holds the events for you to read |

**How the flow works.** You upload a file, then Azure publishes a Blob Created event, then Event Grid matches your subscription, then a message appears in the queue.

**Time.** 40 minutes: Part A 5, B 3, C 8, D 5, E 5, F 8, G 6.

**You need.**

- A computer with a web browser and the sign-in details your trainer gave you for the Azure portal.
- The name of your own resource group and your participant ID, both given by your trainer.
- Notepad, which is already installed on Windows.

**Rules for this lab.**

- Work only inside your own resource group. Do not touch anyone else's resources.
- Do not delete your resource group. A cleanup script removes everything at the end of the day.
- Every name below that contains `<id>` must use your participant ID in lowercase, with no spaces or symbols. For example, if your ID is `p07`, then `stnvt<id>` becomes `stnvtp07`.

**Your names.** Fill in the right-hand column on paper or in Notepad before you begin, so you can copy them as you go.

| Item | Name to use | Your value |
| --- | --- | --- |
| Participant ID | Given by trainer, lowercase |  |
| Resource group | Given by trainer |  |
| Storage account | `stnvt<id>` (3 to 24 lowercase letters and numbers only) |  |
| Blob container | `invoices` |  |
| Queue | `billing-events` |  |
| System topic | `st-billing-<id>` |  |
| Event subscription | `evs-billing-created` |  |
| Dead-letter container (Part G) | `deadletter` |  |

**Open the portal.** In your browser go to https://portal.azure.com and sign in with the account your trainer gave you. If the portal asks to take a tour or shows a welcome page, close it.

## Part A: Create the storage account and the invoices container (5 min)

This storage account is the **source** that will publish events.

1. At the top of the portal, click the search box. Type `Storage accounts` and click **Storage accounts** in the results.
2. Click **+ Create**.
3. On the **Basics** tab, set:
   - **Subscription**: leave the one that is already selected.
   - **Resource group**: click the drop-down and choose **your** resource group. Do not click Create new.
   - **Storage account name**: type `stnvt<id>` using your ID, for example `stnvtp07`. A green tick or no red message means the name is free. If a red message says the name is taken, add one extra digit at the end.
   - **Region**: choose the same region as your resource group. If the resource group region is not in the list, ask your trainer.
   - **Preferred storage type** (if you see it): choose **Azure Blob Storage or Azure Data Lake Storage Gen 2**.
   - **Performance**: **Standard**.
   - **Redundancy**: **Locally-redundant storage (LRS)**.
4. Leave every other tab as it is. Click **Review + create**, wait for the green **Validation passed** message, then click **Create**.
5. Wait about one minute until the page says **Your deployment is complete**, then click **Go to resource**.
6. In the left menu, under **Data storage**, click **Containers**.
7. Click **+ Container**. In **Name** type `invoices`. Leave the access level as **Private**. Click **Create**.

**Checkpoint:** the Containers list shows `invoices`. The storage account must be of kind StorageV2 (general purpose v2) or Blob Storage for Event Grid to work. Creating it as described gives you StorageV2.

## Part B: Create the billing-events queue (3 min)

The queue is the **handler**. In real life the handler would be a function or a Logic App. A queue lets you see exactly what Event Grid delivered, with no code.

1. Stay on your storage account. In the left menu, under **Data storage**, click **Queues**.
2. Click **+ Queue**.
3. In **Queue name** type `billing-events`. The name must be all lowercase.
4. Click **OK**.
5. If the portal shows a message about authentication, such as an authorization error or a link that says **Switch to access key**, click **Switch to access key**. This does not change anything in Azure. It only changes how the portal reads the queue for you.

**Checkpoint:** the Queues list shows `billing-events`, and the queue is empty.

## Part C: Create the event subscription (8 min)

The subscription is the **rule**: it tells Event Grid which events to pick up and where to send them. Azure creates the **system topic** for you at the same time.

1. Go to your storage account. In the left menu click **Events**.
2. In the toolbar at the top of the page click **+ Event Subscription**.
3. On the **Create Event Subscription** page, **Basic** tab, set:
   - **Name**: `evs-billing-created`
   - **Event Schema**: **Event Grid Schema**
   - **System Topic Name**: `st-billing-<id>`, for example `st-billing-p07`. Event Grid creates this topic for the storage account.
4. Under **Event Types**, click **Filter to Event Types**. You will see two ticked boxes, **Blob Created** and **Blob Deleted**. Click **Blob Deleted** to untick it. Only **Blob Created** must stay ticked.
5. Under **Endpoint Details**, click the **Endpoint Type** drop-down and choose **Storage Queues**.
6. Click **Select an endpoint** (the link under Endpoint). A panel opens. Choose the **Subscription**, then your **Storage account** (`stnvt<id>`), then the queue `billing-events`. Click **Confirm selection**.
7. Leave the **Filters**, **Additional features** and **Delivery properties** tabs as they are for now. You will use two of them later.
8. Click **Create** at the bottom.
9. Wait for the notification **Successfully created event subscription**. Then, on the **Events** page, confirm that your subscription appears in the list at the bottom.

**Checkpoint:** the Events page lists `evs-billing-created` with type **Storage Queues**, and in your resource group a new resource of type **Event Grid System Topic** named `st-billing-<id>` now exists.

**If you see an error** saying the resource provider `Microsoft.EventGrid` is not registered, stop and tell your trainer. Registering it needs a permission you do not have in the shared subscription, and the trainer can do it once for everyone.

## Part D: Trigger the event by uploading an invoice (5 min)

**Create the invoice file on your computer.**

1. Click the Windows **Start** button, type `Notepad`, and open **Notepad**.
2. Copy the text below and paste it into Notepad:

```json
{
  "invoiceId": "INV-001",
  "subscriberId": "4471",
  "plan": "Plus-50",
  "amount": 49.90,
  "currency": "EUR",
  "status": "issued"
}
```

3. Click **File**, then **Save as**.
4. In the left pane choose **Desktop**.
5. In **File name** type exactly `invoice-001.json`.
6. Open the **Save as type** drop-down and choose **All files (*.*)**. This stops Notepad from adding `.txt` to the name.
7. Click **Save**. Close Notepad.

**Upload it to the container.**

1. Back in the portal, open your storage account. In the left menu, under **Data storage**, click **Containers**, then click `invoices`.
2. Click **Upload** in the toolbar. A panel opens on the right.
3. Click the **Browse for files** box (or the folder icon). Select `invoice-001.json` on your Desktop and click **Open**.
4. Click **Upload**. Wait for the notification **Successfully uploaded blob(s)**, then close the panel.
5. If the upload fails with an authorization or permission error, look just above the blob list for **Authentication method** and click **Switch to Access key**. Then upload again.

**Checkpoint:** `invoice-001.json` is listed inside `invoices`. Azure has now published a **Blob Created** event, and Event Grid has matched it to your subscription.

## Part E: Read the event in the queue (5 min)

1. Open your storage account. In the left menu, under **Data storage**, click **Queues**, then click `billing-events`.
2. If the list is empty, wait 30 seconds and click **Refresh** at the top. Delivery normally takes a few seconds, but allow up to two minutes before you worry.
3. You should see one message. Click it to open its details panel on the right.
4. Find these fields in the message text and write them down:

| Field | What it tells you | Your value |
| --- | --- | --- |
| `eventType` | What happened. Expect `Microsoft.Storage.BlobCreated` |  |
| `subject` | Which blob. Expect a path ending in `/containers/invoices/blobs/invoice-001.json` |  |
| `eventTime` | When it happened |  |
| `data.url` | The full address of the file |  |
| `data.contentLength` | File size in bytes |  |
| `id` | The unique event ID, the value a handler would use to avoid processing the same event twice |  |

5. Look at the **Dequeue count** column. It shows 0 because nothing has processed the message yet. Viewing a message in the portal does not remove it.

**Checkpoint:** you can explain, in one sentence, how an upload became a queue message. Use the words *source*, *topic*, *subscription* and *handler*.

**Key idea.** The event does not contain the invoice itself. It contains a pointer: the file's address and size. A real billing handler would read that address and fetch the invoice. This is the *thin event* pattern from the session.

## Part F (stretch): Filter so only .json files are delivered (8 min)

Billing should be told about invoices only, not about every file someone drops in the container. You will add a **subject filter** and prove it works.

**Add the filter.**

1. Open your storage account and click **Events** in the left menu.
2. In the list at the bottom, click `evs-billing-created`. Its page opens.
3. Click the **Filters** tab.
4. Tick **Enable subject filtering**.
5. In **Subject Ends With** type `.json`.
6. Leave **Subject Begins With** empty. Click **Save** at the top and wait for the success notification.

**Prepare two test files with Notepad.**

1. Open Notepad. Type `this is not an invoice` and save it on the Desktop as `note.txt` (keep **Save as type** on **Text Documents**).
2. Open `invoice-001.json` again in Notepad, change `INV-001` to `INV-002`, and use **File, Save as** to save it as `invoice-002.json` with **Save as type** set to **All files (*.*)**.

**Run the test.**

1. Go to **Containers**, then `invoices`, and upload `note.txt`.
2. Upload `invoice-002.json` the same way.
3. Go to **Queues**, then `billing-events`, wait about one minute and click **Refresh**.

**Expected result:** the queue holds exactly two messages, one for `invoice-001.json` and one for `invoice-002.json`. There is **no** message for `note.txt`.

**Checkpoint:** you can explain why the `.txt` upload produced no message. Event Grid compared the event's subject, which ends in `/note.txt`, with your filter `.json`, found no match, and did not deliver it. Filtering happens inside Event Grid, so your handler never sees the noise and you pay for no wasted deliveries.

**Extra challenge.** The subject of a blob event looks like `/blobServices/default/containers/invoices/blobs/invoice-002.json`. Which value in **Subject Begins With** would let you accept only events from the `invoices` container and ignore every other container? Write down your answer, then check it with your neighbour.

## Part G (stretch): Dead-lettering and retry settings (6 min)

What if the handler is down? Event Grid retries, and if it still fails it **drops** the event unless you have switched on dead-lettering. Dead-lettering is **off by default**. You will switch it on and look at the retry settings. You will not break anything.

**Create a place for failed events.**

1. Open your storage account, click **Containers**, then **+ Container**.
2. Name it `deadletter`, leave the access level as **Private**, click **Create**.

**Turn on dead-lettering.**

1. Click **Events** in the left menu and open `evs-billing-created`.
2. Click the **Additional features** tab.
3. Tick **Enable dead-lettering**.
4. Click **Select a container** (or the link under Dead-letter storage), choose your subscription, your storage account `stnvt<id>`, and the container `deadletter`. Click **Confirm selection**.
5. Click **Save** and wait for the success notification.

**Read the retry policy.** On the same **Additional features** tab, find the retry policy section and write down:

| Setting | Default you see | Meaning |
| --- | --- | --- |
| Max delivery attempts |  | Event Grid stops retrying after this many tries. The allowed range is 1 to 30 |
| Event time to live (minutes) |  | Event Grid stops retrying after this time. The allowed range is 1 to 1440 minutes |

Event Grid stops at whichever limit it reaches first. Do not change these values.

**Look at the delivery metrics.**

1. In your resource group open the system topic `st-billing-<id>`.
2. In the left menu click **Metrics**.
3. Click the **Metric** drop-down and look at the choices. You should find metrics for published, matched, delivered, failed and dead-lettered events. Names can differ slightly between portal versions.
4. Choose **Delivered Events** (or the closest name) and set the time range to the last hour. You should see deliveries for the uploads you made.

**Checkpoint:** dead-lettering shows as enabled on your subscription, and you can state the two default retry limits.

**Good to know for the real world.**

- Some failures are dead-lettered at once because retrying cannot help, for example a `400` or `413` response from the endpoint.
- A dead-letter blob can appear about five minutes after the failure, so a test that checks too early sees an empty container.
- Event Grid delivers at least once, so a real handler must ignore an event ID it has already processed.

## Done checklist

Tick each box when it is true. Raise your hand and show your trainer when all of Parts A to E are ticked.

- [ ] Storage account `stnvt<id>` exists in my own resource group, with container `invoices`
- [ ] Queue `billing-events` exists
- [ ] Event subscription `evs-billing-created` sends only Blob Created events to the queue
- [ ] Uploading `invoice-001.json` produced a message in the queue
- [ ] I wrote down the `eventType`, `subject` and `id` from the message
- [ ] Stretch F: the `.txt` upload produced no message, and the queue holds exactly two
- [ ] Stretch G: dead-lettering is enabled and I know the default retry limits

## If something goes wrong

| What you see | Likely cause | What to do |
| --- | --- | --- |
| Storage account name is not available | Names are global across Azure | Add one more digit to your ID suffix |
| Error mentions `Microsoft.EventGrid` not registered | Provider not enabled in the subscription | Tell your trainer |
| The subscription option for Storage Queues is missing | Wrong storage account type | Check the account is StorageV2 (general purpose v2) |
| Upload or queue view shows an authorization error | Portal is using your sign-in, not the access key | Click **Switch to Access key** and try again |
| Saved file is `invoice-001.json.txt` | Notepad added `.txt` | Save again with **Save as type** set to **All files (*.*)** |
| Queue is empty after two minutes | Subscription not created, wrong event type, or file uploaded to another account | Check the Events page lists your subscription, **Blob Created** is ticked, and the blob is in your own `invoices` container |
| Queue is empty after Part F | Filter has a typo or a leading space | Reopen the subscription, **Filters** tab, and retype `.json` |
| Resource not found or no permission in your resource group | Wrong resource group selected | Check the resource group name at the top of the page matches yours |

## Think and discuss (5 min)

Write one or two sentences for each. Your trainer will take answers at the end.

1. Which Azure resource published the event, and which resource received it?
2. You want the loyalty team to also react to every new invoice. What do you add, and does the storage account change?
3. What is the difference between an event and a message, using this lab as the example?
4. Why is dead-lettering worth switching on for billing events, given that it is off by default?
5. Suppose Event Grid delivers the same invoice event twice. What should a real billing handler check before charging?

## Sources

Step names follow these Microsoft Learn pages, consulted on 2026-10-16. Portal labels can change slightly between releases.

- [Quickstart: Route Blob storage events to a web endpoint (Azure portal)](https://learn.microsoft.com/azure/event-grid/blob-event-quickstart-portal)
- [Subscribe to events through the portal](https://learn.microsoft.com/azure/event-grid/subscribe-through-portal)
- [Reacting to Blob storage events](https://learn.microsoft.com/azure/storage/blobs/storage-blob-event-overview)
- [Set dead-letter location and retry policy](https://learn.microsoft.com/azure/event-grid/manage-event-delivery)
