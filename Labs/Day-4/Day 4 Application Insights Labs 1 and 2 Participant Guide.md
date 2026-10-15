# Day 4 Application Insights Labs 1 and 2: Participant Guide

Oct 15, 2026 · @Nived Varma

## Before you start

All three parts run in your own resource group and use the Azure portal only. Part A deploys the NovaTel portal and billing Function there; Labs 1 and 2 then add monitoring and diagnose a problem. Your trainer gives you the values for the placeholders below. Write them in the last column, then replace every placeholder in this guide with your own value. Use lowercase letters and digits for your participant ID.

| Placeholder | What it is | Your value |
| --- | --- | --- |
| `[id]` | Your participant ID |  |
| `<RG_NAME>` | Your resource group, already created for you (usually `rg-nvtl-oct26-[id]`) |  |
| `<REGION>` | The region of that group; create everything in the same region |  |
| `<AI_NAME>` | The Application Insights resource that already exists in your group |  |
| `<PORTAL_PACKAGE_URL>` | Link to the portal package, from your trainer |  |
| `<FUNCTION_PACKAGE_URL>` | Link to the billing Function package, from your trainer |  |

Names you create in Part A, all lowercase:

| Resource | Name |
| --- | --- |
| Storage account | `stnvtl[id]` |
| Blob container | `invoices` |
| Function app | `func-nvtl-billing-[id]` |
| App Service plan | `asp-nvtl-[id]` |
| Web app (portal) | `app-nvtl-portal-[id]` |

|  | Part A (25 min) | Lab 1 (40 min) | Lab 2 (40 min) |
| --- | --- | --- | --- |
| Goal | Deploy the portal, Function and storage | Both apps send telemetry | Detect an outage, then find a slow dependency |
| Done when | `/invoice/INV-1001` on your portal returns an invoice | Both apps appear on the application map and one trace shows a request with its dependency | You name the slow dependency and its p95, and your alert has fired once |

Telemetry takes two to five minutes to appear. If a screen is empty, generate more traffic and wait before you change anything.

## Part A: Deploy the NovaTel starter in your resource group (25 min)

You will create a storage account, the billing Function app and the portal web app in `<RG_NAME>`, then load the prepared code into both from the package links your trainer gave you. Your Application Insights resource `<AI_NAME>` already exists; leave it alone until Lab 1. Keep a Notepad window open for the values you copy.

1. **Check your group.** Search for **Resource groups** > `<RG_NAME>`. Confirm `<AI_NAME>` is listed and note the region. Create everything below in that region.
2. **Create the storage account.** In the group select **Create** > search **Storage account**. Name `stnvtl[id]`, Performance **Standard**, Redundancy **LRS**. Select **Review + create**, then **Create**, then **Go to resource**.
3. **Create the container.** Open **Data storage** > **Containers** > **+ Container**. Name `invoices`, public access **Private**, then **Create**.
4. **Copy the connection string (checkpoint at minute 8).** Open **Security + networking** > **Access keys** > **Show**. Copy the **Connection string** under key1 into Notepad as `STORAGE_CONN`.
5. **Create the Function app.** In the group select **Create** > **Function App**. Hosting option **Consumption** (not Flex Consumption), name `func-nvtl-billing-[id]`, runtime stack **.NET**, version **8 (LTS), isolated worker model**, operating system **Windows**. On the **Storage** tab pick `stnvtl[id]`. On the **Monitoring** tab set **Enable Application Insights** to **No**; you connect it in Lab 1. **Review + create**, then **Create**.
6. **Load the code into the Function app.** Open it > **Settings** > **Environment variables** > **App settings** > **Add**, twice: `INVOICE_STORAGE` with the value `STORAGE_CONN`, and `WEBSITE_RUN_FROM_PACKAGE` with the value `<FUNCTION_PACKAGE_URL>`. Select **Apply**, then **Confirm**. Then **Overview** > **Restart**.
7. **Check the function appears (checkpoint at minute 14).** On **Overview**, select **Refresh** until the **Functions** list shows `GenerateInvoice`. This can take a minute or two.
8. **Copy the function key.** Open **Functions** > **App keys**, show the host key named `default`, and copy it into Notepad as `FUNC_KEY`.
9. **Create the portal web app.** In the group select **Create** > **Web App**. Name `app-nvtl-portal-[id]`, publish **Code**, runtime stack **.NET 8 (LTS)**, operating system **Linux**, same region. Create a new plan `asp-nvtl-[id]` with pricing plan **Basic B1**. On the **Monitoring** tab set **Enable Application Insights** to **No**. **Review + create**, then **Create**.
10. **Connect the portal to the Function and load its code.** Open the web app > **Settings** > **Environment variables** > **App settings** > **Add** three settings: `BILLING_FUNCTION_URL` with `https://func-nvtl-billing-[id].azurewebsites.net` (use the exact URL shown on the Function app's Overview if it differs), `BILLING_FUNCTION_KEY` with `FUNC_KEY`, and `WEBSITE_RUN_FROM_PACKAGE` with `<PORTAL_PACKAGE_URL>`. Select **Apply**, then **Confirm**.
11. **Test it (checkpoint at minute 22).** On the web app's **Overview**, open the **Default domain**. Add `/health` to the address: you should see `ok`. Then open `/invoice/INV-1001`: you should see an invoice in JSON. The first call can take up to a minute while the apps wake up.
12. **Confirm the write.** In `stnvtl[id]` open **Containers** > `invoices`. A file named `INV-1001.json` should be there.

**Done when** `/invoice/INV-1001` on your portal returns an invoice and the file exists in Blob Storage.

**If it doesn't work**

- `/health` fails: the code did not load. Check `WEBSITE_RUN_FROM_PACKAGE` is spelled exactly, the link was pasted whole, then restart the web app.
- `/invoice/...` returns an error: check `BILLING_FUNCTION_URL` has no trailing spaces and that `FUNC_KEY` is the `default` host key.
- A `504` or timeout message at this stage means the Function is cold or the storage connection string is wrong. Wait a minute, then re-check `INVOICE_STORAGE`.
- The package link expired or returns an error: tell your trainer.

## Lab 1: Instrument the NovaTel apps (40 min)

Part A left you with a working portal and billing Function that send no telemetry yet. You will connect both to the Application Insights resource `<AI_NAME>` that already exists in `<RG_NAME>`, so a request from the portal can be followed into the billing Function and on to storage.

1. **Find your resource (checkpoint at minute 5).** Open `<RG_NAME>` > `<AI_NAME>`. On **Overview**, copy the **Connection String** into Notepad and check there are no spaces before or after it.
2. **Connect the portal.** Open web app `app-nvtl-portal-[id]` > **Monitoring** > **Application Insights** > **Turn on**. Select `<AI_NAME>`, keep the recommended settings, select **Apply**, and accept the restart.
3. **Connect the Function app.** Open `func-nvtl-billing-[id]` > **Settings** > **Environment variables** > **App settings** > **Add**. Name `APPLICATIONINSIGHTS_CONNECTION_STRING`, value the string from step 1. Select **Apply**, then **Confirm**. The app restarts.
4. **Generate traffic (checkpoint at minute 25).** Open your portal address, then `/invoice/INV-1001` and `/invoice/INV-1002`. Refresh them about 20 times over two minutes. Each page calls the Function, which writes to Blob Storage.
5. **Watch it live.** Open `<AI_NAME>` > **Investigate** > **Live Metrics**. You should see incoming requests within seconds. Live Metrics is real time and is not stored.
6. **Find your requests (checkpoint at minute 35).** Open **Transaction search**, set the time range to the last hour, and select one request. Open **Application map** and confirm the portal, the Function and the storage account appear.

**Done when** both apps are on the application map and one trace shows a request together with its dependency.

**If it doesn't work**

- Live Metrics shows no servers: restart the app, wait two minutes, generate traffic again.
- The application map is empty: wait up to five minutes, and check the resource group and region match your apps.
- Nothing from the Function: check the setting name is spelled exactly as above and that you selected **Confirm**.

**Early finishers:** set a cloud role name for each app so the map tells them apart, using the `OTEL_SERVICE_NAME` app setting, or open **Performance** and name the slowest operation.

## Lab 2: Alert on it, then diagnose it (40 min)

The portal looks slow and some invoices fail. Detect it with a probe and an alert, then use KQL to find the real cause. The cause is different from the one in the trainer's demo, so do not assume it is the database.

1. **Availability test (checkpoint at minute 10).** In `<AI_NAME>` open **Investigate** > **Availability** > **Add Standard test**. Name `portal-health-[id]`, URL `https://app-nvtl-portal-[id].azurewebsites.net/health`, frequency 5 minutes, at least three locations, success when the status code is 200. Leave the alert checkbox on and select **Create**.
2. **Action group.** Open **Monitor** > **Alerts** > **Action groups** > **Create**. Name `ag-nvtl-[id]`, resource group `<RG_NAME>`, add an **Email** notification to your own address.
3. **Failed-requests alert (checkpoint at minute 15).** In `<AI_NAME>` open **Alerts** > **Create** > **Alert rule**. Signal **Failed requests**, aggregation Count, greater than `3`, checked every 1 minute over the last 5 minutes. Select action group `ag-nvtl-[id]`, severity 2, name `billing-failures-[id]`, and make sure it is Enabled.
4. **Switch on the problem (checkpoint at minute 20).** Open Function app `func-nvtl-billing-[id]` > **Environment variables** > **Add**. Name `BILLING_FAULT_MODE`, value `storage-latency`. Select **Apply** and **Confirm**.
5. **Generate traffic.** Open `/invoice/INV-1001`, `/invoice/INV-1002` and `/invoice/INV-1003` on your portal and refresh them about 20 times over three or four minutes. Each call now takes several seconds and some fail.
6. **Find the cause (checkpoint at minute 35).** Open `<AI_NAME>` > **Logs**, set the time picker to **Last hour**, and run the queries from the KQL pack below, one at a time.
7. **Say it in one sentence (minute 40).** Use this shape: *"\[Operation\] is slow because \[dependency\] has a p95 of \[value\], with \[failure type\] failures."*

**Done when** you can name the slow dependency and its p95, and your alert has fired once. Fired alerts always show under **Monitor** > **Alerts**, even if the email lands in spam.

**Waiting for the probe?** It needs at least one interval before results appear. Start the KQL queries on your Lab 1 data in the meantime.

**Hints, only when you are stuck:** 1) open **Failures** and group by operation; 2) sort the dependencies query by p95; 3) open one failed request in **Transaction search** and read the end-to-end trace.

## KQL pack

Run these in `<AI_NAME>` > **Logs**. Select one query and press **Run** on its own. These are the Application Insights table names; in a Log Analytics workspace the same tables are `AppRequests`, `AppDependencies` and so on, with some column names differing.

**1. Which operation fails, and with what code?**

```kusto
requests
| where timestamp > ago(1h)
| where success == false
| summarize failures = count() by name, resultCode
| order by failures desc
```

**2. Which operation is slowest at the tail?**

```kusto
requests
| where timestamp > ago(1h)
| summarize p95_ms = percentile(duration, 95), calls = count() by name
| order by p95_ms desc
```

**3. Which dependency is slowest, and does it fail?**

```kusto
dependencies
| where timestamp > ago(1h)
| summarize p95_ms = percentile(duration, 95), calls = count(), failures = countif(success == false) by type, target, name
| order by p95_ms desc
```

**4. Prove it: link failed requests to their dependency calls.**

```kusto
requests
| where timestamp > ago(1h) and success == false
| join kind=inner (dependencies | where timestamp > ago(1h)) on operation_Id
| project timestamp, operation = name, dependency = name1, target, depResult = resultCode1, depMs = duration1
| order by timestamp desc
| take 20
```

**5. Chart it.** Add `| render timechart` after a `summarize ... by bin(timestamp, 5m)` to see when the problem began.

| Operator | Use it to |
| --- | --- |
| `where` | keep only matching rows |
| `summarize` | count or aggregate by group |
| `percentile(duration, 95)` | see tail latency that an average hides |
| `join` | link tables through `operation_Id` |
| `project` | choose which columns to show |
| `bin` and `render` | group by time and draw a chart |

Duration is in milliseconds. If a query returns nothing, check the time picker first, then wait a few minutes for telemetry to arrive.

## Trainer notes: gaps to close before delivery

The guide above is ready to hand out, but the labs only work if three things exist that I cannot see from here: the baseline apps, the fault switch in the Function code, and an anonymous health path. Nothing here has been run in Azure.

**Portal-only deployment: what you prepare once.** Participants load code through `WEBSITE_RUN_FROM_PACKAGE`, so you publish two zip files and hand out read-only links. `scripts/publish-packages.ps1` builds both zips, uploads them to a storage account in `rg-nvtl-platform`, and prints the two links as `<PORTAL_PACKAGE_URL>` and `<FUNCTION_PACKAGE_URL>`. The links expire after two days, so generate them the evening before and share them with the placeholder values (group name, region, Application Insights name) on one slide or handout.

**Timing.** Part A adds about 25 minutes, so the Application Insights block becomes about 105 minutes of lab. Either move Part A to the start of the day, or let participants who fall behind use your pre-deployed group: `deploy-day4-ai.ps1` still works as the fallback and builds the same resources.

**Portal limits to know.** The portal's Code + Test tab cannot run compiled .NET isolated functions, so the guide generates traffic through the portal's invoice pages instead. If the portal's Function creation page offers only Flex Consumption in your tenant, switch the hosting choice to Consumption, or ask for a Premium or App Service plan; Flex Consumption does not use `WEBSITE_RUN_FROM_PACKAGE`.

| Gap | Why it matters | What to do |
| --- | --- | --- |
| Baseline apps in every participant group | Groups are deleted nightly and each day stands alone, so the portal and billing Function from Day 3 will not be there | Deploy both into each `rg-nvtl-oct26-[id]` with deploy-day4-ai.ps1 the evening before, and rehearse one group end to end |
| `BILLING_FAULT_MODE` is now in the starter Function | Lab 2 step 4 only works if the Function reads this setting | Run set-fault.ps1 -On on a test group, and confirm KQL query 3 shows the storage call as slowest |
| Portal is behind Entra sign-in | The availability test would see a redirect or 401 and fail for the wrong reason | Add an anonymous `/health` endpoint, or exclude that path from authentication |
| Portal may already reference Application Insights packages | The App Service switch can back off if the app is already instrumented | Check the project references, and if present, remove them from the baseline |
| 40 minutes is tight for Lab 2 | Seven steps plus waiting for the probe | Pre-create the action group per participant, or let early finishers skip step 2 and share one |

**Fault switch.** It is now in the starter Function (`FaultPolicy.cs`). With `BILLING_FAULT_MODE=storage-latency`, the Blob call gets a 3–8 second delay and a 30% simulated timeout, so the storage dependency shows a high p95 with intermittent failures. The starter code is untested until you build it once.

This delivers the per-participant seeded issue as one app setting, which closes the earlier open item about how Lab 2's issue reaches each person. Because every participant flips their own setting, nobody affects anyone else.

**Expected answer:** the storage dependency used by invoice generation has a high p95 and timeout failures, and the portal only looks slow because it waits on the Function.

**Cost note:** each Standard availability test runs from several locations every five minutes, so a full class adds up; delete the tests with the group at the end of the day.
