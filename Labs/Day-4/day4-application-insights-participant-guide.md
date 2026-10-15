# Day 4 Labs: Application Insights on the NovaTel platform

Participant guide. Everything is done in the Azure portal and in your own resource group. You deploy the NovaTel portal and billing Function yourself (Part A), connect them to Application Insights (Lab 1), then detect and diagnose a problem (Lab 2).

| Part | Time | Done when |
| --- | --- | --- |
| A. Deploy the NovaTel starter | 25 min | `/invoice/INV-1001` on your portal returns an invoice |
| Lab 1. Instrument the apps | 40 min | Both apps appear on the application map and one trace shows a request with its dependency |
| Lab 2. Alert on it, then diagnose it | 40 min | You name the slow dependency and its p95, and your alert has fired once |

Telemetry takes two to five minutes to appear. If a screen is empty, generate more traffic and wait before you change anything.

## Before you start

You need two files from this repository's `packages/` folder. Download them to your Windows machine and do not unzip them.

- `func.zip` (the billing Function)
- `web.zip` (the portal)

Your trainer gives you the values below. Write them in the last column, then replace every placeholder in this guide with your own value. Use lowercase letters and digits only for your participant ID.

| Placeholder | What it is | Your value |
| --- | --- | --- |
| `[id]` | Your participant ID | |
| `<RG_NAME>` | Your resource group, already created for you (usually `rg-nvtl-oct26-[id]`) | |
| `<REGION>` | The region of that group; create everything in the same region | |
| `<AI_NAME>` | The Application Insights resource that already exists in your group | |

Names you create in Part A, all lowercase:

| Resource | Name |
| --- | --- |
| Storage account | `stnvtl[id]` |
| Blob container | `invoices` |
| Function app | `func-nvtl-billing-[id]` |
| App Service plan | `asp-nvtl-[id]` |
| Web app (portal) | `app-nvtl-portal-[id]` |

Keep a Notepad window open for the values you copy. Use synthetic data only.

---

## Part A: Deploy the NovaTel starter (25 min)

Your Application Insights resource `<AI_NAME>` already exists. Leave it alone until Lab 1.

1. **Check your group.** Search for **Resource groups** > `<RG_NAME>`. Confirm `<AI_NAME>` is listed and note the region.
2. **Create the storage account.** In the group select **Create** > search **Storage account**. Name `stnvtl[id]`, Performance **Standard**, Redundancy **LRS**. Select **Review + create**, then **Create**, then **Go to resource**.
3. **Create the container.** Open **Data storage** > **Containers** > **+ Container**. Name `invoices`, public access **Private**, then **Create**.
4. **Copy the connection string.** Open **Security + networking** > **Access keys** > **Show**. Copy the **Connection string** under key1 into Notepad as `STORAGE_CONN`. *Checkpoint at minute 8: storage account and container exist.*
5. **Create the Function app.** In the group select **Create** > **Function App**. Hosting option **Consumption** (not Flex Consumption), name `func-nvtl-billing-[id]`, runtime stack **.NET**, version **8 (LTS), isolated worker model**, operating system **Windows**. On the **Storage** tab pick `stnvtl[id]`. On the **Monitoring** tab set **Enable Application Insights** to **No**; you connect it in Lab 1. Select **Review + create**, then **Create**, then **Go to resource**.
6. **Give the Function its storage setting.** Open **Settings** > **Environment variables** > **App settings** > **Add**. Name `INVOICE_STORAGE`, value `STORAGE_CONN`. Select **Apply**, then **Confirm**.
7. **Upload the Function code.**
   1. Open **Development Tools** > **Advanced Tools** > **Go**. A new tab opens; sign in if asked.
   2. In the address bar, change the end of the address to `/ZipDeployUI` (for example `https://func-nvtl-billing-[id].scm.azurewebsites.net/ZipDeployUI`).
   3. Drag `func.zip` from File Explorer onto the page and wait for the log to say the deployment succeeded.
   4. Back in the portal, open the Function app **Overview** > **Restart**.
8. **Check the function appears.** On **Overview**, select **Refresh** until the **Functions** list shows `GenerateInvoice`. This can take a minute or two. *Checkpoint at minute 14.*
9. **Copy the function key.** Open **Functions** > **App keys**, show the host key named `default`, and copy it into Notepad as `FUNC_KEY`.
10. **Create the portal web app.** In the group select **Create** > **Web App**. Name `app-nvtl-portal-[id]`, publish **Code**, runtime stack **.NET 8 (LTS)**, operating system **Linux**, same region. Create a new plan `asp-nvtl-[id]` with pricing plan **Basic B1**. On the **Monitoring** tab set **Enable Application Insights** to **No**. Select **Review + create**, then **Create**, then **Go to resource**.
11. **Connect the portal to the Function.** Open **Settings** > **Environment variables** > **App settings** > **Add** two settings: `BILLING_FUNCTION_URL` with `https://func-nvtl-billing-[id].azurewebsites.net` (use the exact URL on the Function app's Overview if it differs) and `BILLING_FUNCTION_KEY` with `FUNC_KEY`. Select **Apply**, then **Confirm**.
12. **Upload the portal code.** Open **Development Tools** > **Advanced Tools** > **Go**, change the address ending to `/ZipDeployUI`, drag `web.zip` onto the page, and wait for the success message. Then **Overview** > **Restart**.
13. **Test it.** On the web app **Overview**, open the **Default domain**. Add `/health` to the address: you should see `ok`. Then open `/invoice/INV-1001`: you should see an invoice in JSON. The first call can take up to a minute while the apps wake up. *Checkpoint at minute 22.*
14. **Confirm the write.** In `stnvtl[id]` open **Containers** > `invoices`. A file named `INV-1001.json` should be there.

**Done when** `/invoice/INV-1001` on your portal returns an invoice and the file exists in Blob Storage.

**If it doesn't work**

- The zip page asks for a password or shows an access error: in the app, open **Settings** > **Configuration** > **General settings** and turn **SCM Basic Auth Publishing Credentials** on, then reload the page.
- The upload says the app is read-only: open the app's **App settings** and delete any setting called `WEBSITE_RUN_FROM_PACKAGE`, then upload again.
- `/health` fails: restart the web app and wait a minute. Check the upload log reported success.
- `/invoice/...` returns an error: check `BILLING_FUNCTION_URL` has no trailing spaces and that `FUNC_KEY` is the `default` host key.
- A `504` or timeout message at this stage: the Function is cold, or `INVOICE_STORAGE` is wrong. Wait a minute, then re-check it.

---

## Lab 1: Instrument the NovaTel apps (40 min)

Part A left you with a working portal and billing Function that send no telemetry yet. Connect both to the Application Insights resource `<AI_NAME>` in `<RG_NAME>`, so a request from the portal can be followed into the Function and on to storage.

1. **Find your resource.** Open `<RG_NAME>` > `<AI_NAME>`. On **Overview**, copy the **Connection String** into Notepad and check there are no spaces before or after it. *Checkpoint at minute 5.*
2. **Connect the portal.** Open web app `app-nvtl-portal-[id]` > **Monitoring** > **Application Insights** > **Turn on**. Select `<AI_NAME>`, keep the recommended settings, select **Apply**, and accept the restart.
3. **Connect the Function app.** Open `func-nvtl-billing-[id]` > **Settings** > **Environment variables** > **App settings** > **Add**. Name `APPLICATIONINSIGHTS_CONNECTION_STRING`, value the string from step 1. Select **Apply**, then **Confirm**. The app restarts.
4. **Generate traffic.** Open your portal address, then `/invoice/INV-1001` and `/invoice/INV-1002`. Refresh them about 20 times over two minutes. Each page calls the Function, which writes to Blob Storage. *Checkpoint at minute 25.*
5. **Watch it live.** Open `<AI_NAME>` > **Investigate** > **Live Metrics**. You should see incoming requests within seconds. Live Metrics is real time and is not stored.
6. **Find your requests.** Open **Transaction search**, set the time range to the last hour, and select one request. Open **Application map** and confirm the portal, the Function and the storage account appear. *Checkpoint at minute 35.*

**Done when** both apps are on the application map and one trace shows a request together with its dependency.

**If it doesn't work**

- Live Metrics shows no servers: restart the app, wait two minutes, generate traffic again.
- The application map is empty: wait up to five minutes, and check the resource group and region match your apps.
- Nothing from the Function: check the setting name is spelled exactly as above and that you selected **Confirm**.

**Early finishers:** set a cloud role name for each app so the map tells them apart, using the `OTEL_SERVICE_NAME` app setting, or open **Performance** and name the slowest operation.

---

## Lab 2: Alert on it, then diagnose it (40 min)

The portal looks slow and some invoices fail. Detect it with a probe and an alert, then use KQL to find the real cause. The cause is not the database, so do not assume it is.

1. **Availability test.** In `<AI_NAME>` open **Investigate** > **Availability** > **Add Standard test**. Name `portal-health-[id]`, URL `https://app-nvtl-portal-[id].azurewebsites.net/health`, frequency 5 minutes, at least three locations, success when the status code is 200. Leave the alert checkbox on and select **Create**. *Checkpoint at minute 10.*
2. **Action group.** Open **Monitor** > **Alerts** > **Action groups** > **Create**. Name `ag-nvtl-[id]`, resource group `<RG_NAME>`, add an **Email** notification to your own address.
3. **Failed-requests alert.** In `<AI_NAME>` open **Alerts** > **Create** > **Alert rule**. Signal **Failed requests**, aggregation Count, greater than `3`, checked every 1 minute over the last 5 minutes. Select action group `ag-nvtl-[id]`, severity 2, name `billing-failures-[id]`, and make sure it is Enabled. *Checkpoint at minute 15.*
4. **Switch on the problem.** Open Function app `func-nvtl-billing-[id]` > **Environment variables** > **Add**. Name `BILLING_FAULT_MODE`, value `storage-latency`. Select **Apply** and **Confirm**. *Checkpoint at minute 20.*
5. **Generate traffic.** Open `/invoice/INV-1001`, `/invoice/INV-1002` and `/invoice/INV-1003` on your portal and refresh them about 20 times over three or four minutes. Each call now takes several seconds and some fail.
6. **Find the cause.** Open `<AI_NAME>` > **Logs**, set the time picker to **Last hour**, and run the queries from the KQL pack below, one at a time. *Checkpoint at minute 35.*
7. **Say it in one sentence.** Use this shape: *"[Operation] is slow because [dependency] has a p95 of [value], with [failure type] failures."*

**Done when** you can name the slow dependency and its p95, and your alert has fired once. Fired alerts always show under **Monitor** > **Alerts**, even if the email lands in spam.

**Waiting for the probe?** It needs at least one interval before results appear. Start the KQL queries on your Lab 1 data in the meantime.

**Hints, only when you are stuck:** 1) open **Failures** and group by operation; 2) sort the dependencies query by p95; 3) open one failed request in **Transaction search** and read the end-to-end trace.

---

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

## Clean up

Your trainer deletes the resource groups at the end of the day. Nothing to do.
