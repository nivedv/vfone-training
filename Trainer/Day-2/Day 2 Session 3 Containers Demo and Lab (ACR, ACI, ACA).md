# Day 2 Session 3: Containers Demo and Lab (ACR, ACI, ACA)

Oct 14, 2026 · @Nived Varma

## How to use this doc

This one document runs both the trainer demos and your own lab, and you can repeat it later on your own. Docker is not installed on the sandbox and you do not need it: images are built inside Azure with Azure Container Registry (ACR) Tasks, started from Azure Cloud Shell in your browser. Everything else is done in the Azure portal.

### Labels used in every step

| Label | Who | What to do |
| --- | --- | --- |
| **Trainer demo** | Trainer | Watch. Do not click yet. The trainer works in the trainer resource group. |
| **Follow along** | Everyone | Repeat the same steps in your own resource group straight after the demo. |
| **Your lab** | Everyone | Work alone from the task and the Check table. Open the Hint only if stuck. |
| **Pair task** | Two people | Swap seats and break or fix each other's deployment. |
| **Expect** | Everyone | What you should see. If you do not see it, stop and use Troubleshooting. |
| **Stretch** | Fast finishers | Optional extra. Skip if short on time. |

### Replace these values first

Wherever you see angle brackets, swap in your own value. Write yours down before you start.

| Placeholder | Meaning | Example |
| --- | --- | --- |
| `<id>` | Your short lowercase ID from the trainer, letters and digits only | `nv07` |
| `<rg>` | Your resource group | `rg-nvtl-oct26-nv07` |
| `<region>` | The region of your resource group, as the trainer tells you | shown on your resource group page |
| `<acr>` | Your registry name, 5 to 50 lowercase letters and digits, no hyphens | `acrnovatelnv07` |
| `<image>` | Image repository name | `novatel-billing` |

Resource names in this doc follow one pattern so the cleanup script and the trainer can find them: registry `acrnovatel<id>`, Container Apps environment `cae-novatel-<id>`, container app `ca-billing-<id>`, container instance `aci-billing-<id>`.

### Rules for the shared subscription

1. Work only in your own resource group. Never open or edit another person's resources.
2. Every resource you create must carry the tags `course = novatel-oct26`, `day = 2` and `owner = <id>`. The end-of-day cleanup deletes by these tags.
3. Do not create a resource group. If you cannot see yours, ask the trainer.
4. Use only the synthetic data in this doc. Never put real customer data in an image or a container.
5. Do not use the Docker command, `docker build` or `docker run`. They will not work here.
6. Delete or stop what the lab tells you to. Container Apps and Container Instances that keep running cost money.

### Practising later

The lab is independent of Days 1 and 3, so you can repeat it in any subscription where you are allowed to create a registry. Start from **Prepare**, use a new `<id>`, and finish with the cleanup section.

### What to hand in

A short note with: your registry login server, the image tag you built, the public address of your container app, the number of replicas you saw under load, and one thing that failed and how you fixed it.

## Scenario and outcomes

**Scenario.** NovaTel's billing team has a small billing microservice that returns a customer's plan, usage and a calculated charge. Today it runs on a shared server. You will package it as a container image, store it in a private registry, run it on demand, then run it as a service that scales with traffic, without Docker on your machine.

The service is a tiny .NET 8 minimal API. You create its few files by pasting from this doc, so no download is needed. It holds 20 synthetic customers in memory and exposes `/health`, `/api/customers/{id}/bill` and a deliberately slow `/api/load` endpoint that you use to trigger scaling.

By the end you can:

1. Build a container image in Azure Container Registry with `az acr build`, and see it under **Repositories** in the portal.
2. Explain why a registry login server, repository and tag identify an image.
3. Run the image once as an Azure Container Instance (ACI) and read its logs.
4. Deploy the same image as an Azure Container App (ACA) with external ingress.
5. Add an HTTP scale rule, generate load, and watch replicas increase and then return to the minimum.
6. Choose between ACI, ACA and App Service for a NovaTel workload and justify it.

### Timing

| Block | Minutes |
| --- | --- |
| Prepare (registry, Cloud Shell, source files) | 20 |
| Demo 1 and Lab 1: build and verify the image | 40 |
| Demo 2 and Lab 2: ACI, then Container Apps | 45 |
| Lab 3: scale, break cards, compare | 40 |
| Quiz and cleanup | 15 |

About two and a half hours with a break. Prepare and the first build can run while the trainer talks.

### Which service for which job

|  | Azure Container Instances | Azure Container Apps | App Service (container) |
| --- | --- | --- | --- |
| Best for | Short jobs, tests, one-off tasks | Microservices and event-driven APIs | Web apps and APIs you want managed like a classic app |
| Scaling | None built in. You create more groups | Built in: HTTP, queue and custom rules, can scale to zero | Scale up and out by plan, autoscale rules |
| Revisions and traffic splitting | No | Yes | Deployment slots |
| Ingress and TLS | Public IP or DNS label, no managed TLS | Managed HTTPS ingress | Managed HTTPS and custom domains |
| Billing | Per second for CPU and memory while running | Per second while running, can be zero at idle | Per plan, whether busy or idle |
| NovaTel example | Run a one-off invoice rebuild | The billing API that peaks at month end | The customer self-care web app |

The pricing and feature statements above are summaries. Confirm limits in the portal before you quote them to a project.

## Prepare (everyone, about 20 minutes)

The trainer does this once in `rg-nvtl-oct26-trainer` with ID `nvtrain`. You do it in your own resource group. The registry takes a couple of minutes to deploy, so start step 1 first.

### Step 1: Create your container registry (portal)

1. Sign in to the Azure portal. Open **Resource groups** and click your resource group `<rg>`. Note its **Location**. This is your `<region>`.
2. Click **Create** at the top of the resource group. Search for **Container Registry** and click **Create**.
3. On the **Basics** tab set:
   - **Subscription**: the shared subscription.
   - **Resource group**: `<rg>`.
   - **Registry name**: `acrnovatel<id>`.
   - **Location**: your `<region>`.
   - **Pricing plan**: **Basic**.
4. Open the **Tags** tab and add `course = novatel-oct26`, `day = 2`, `owner = <id>`.
5. Click **Review + create**, then **Create**. When it says deployment is complete, click **Go to resource**.
6. In the left menu open **Overview** and copy the **Login server**. It looks like `acrnovatel<id>.azurecr.io`.
7. In the left menu open **Settings**, then **Access keys**, and turn **Admin user** on.

**Expect:** the registry page shows **Provisioning state: Succeeded** and an empty **Repositories** list.

**Login server name.** Newer registries can add a short hash to the name, for example `acrnovatelnv07-abc123.azurecr.io`. Always copy the exact **Login server** from the Overview page and never type it from memory. Later steps read it with a command, so you do not need to edit it by hand.

**Why admin user?** It lets ACI and ACA pull your image with a simple user name and password in a short lab. In production NovaTel would use a managed identity with the `AcrPull` role and keep admin user off. You will see where that choice appears in Lab 2.

### Step 2: Open Cloud Shell

1. In the top bar of the portal click the **Cloud Shell** icon (`>_`).
2. Choose **Bash**.
3. If it asks for storage, choose the option for **no storage account required** and select the shared subscription. If that option is not offered, ask the trainer. Do not create a storage account of your own without the right tags.
4. Wait for the prompt. Paste into Cloud Shell with **Ctrl+Shift+V** (or right-click, **Paste**).

Set your values once. Replace `nv07` with your ID and `<rg>` with your resource group name:

```bash
ID=nv07
RG=rg-nvtl-oct26-$ID
ACR=acrnovatel$ID
IMAGE=novatel-billing
TAG=v1
az account show --query "{subscription:name, user:user.name}" -o table
az group show -n $RG --query "{name:name, location:location}" -o table
```

**Expect:** your subscription name and your resource group with its location. If Cloud Shell closes after idling, open it again and repeat this block.

### Step 3: Create the billing microservice source files

You will paste four small files. Each command writes a file in a new folder. Copy each block completely, including the last line (`EOF`).

```bash
mkdir -p ~/novatel-billing && cd ~/novatel-billing

cat > NovaTel.Billing.csproj <<'EOF'
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
EOF

cat > .dockerignore <<'EOF'
bin/
obj/
EOF

cat > Dockerfile <<'EOF'
# Stage 1: build with the full SDK
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY NovaTel.Billing.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

# Stage 2: run on the small ASP.NET runtime image
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "NovaTel.Billing.dll"]
EOF
```

Now the application code:

```bash
cat > Program.cs <<'EOF'
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Plan catalogue: monthly fee, included GB, price per extra GB (EUR)
var plans = new Dictionary<string, Plan>
{
    ["Basic"] = new("Basic", 15m, 5, 2.00m),
    ["Plus"]  = new("Plus", 25m, 20, 1.50m),
    ["Pro"]   = new("Pro", 40m, 50, 1.00m)
};

// 20 synthetic customers, held in memory. No database, no real data.
var customers = Enumerable.Range(1, 20)
    .Select(i => new Customer(
        $"c-{1000 + i}",
        $"Customer {i:00}",
        i % 3 == 0 ? "Pro" : i % 2 == 0 ? "Plus" : "Basic",
        i % 2 == 0 ? "IN" : "UK",
        Math.Round(2.5 * i, 1)))
    .ToDictionary(c => c.Id);

var version = Environment.GetEnvironmentVariable("APP_VERSION") ?? "v1";

app.MapGet("/", () => Results.Ok(new { service = "novatel-billing", version, host = Environment.MachineName }));

app.MapGet("/health", () => Results.Ok(new { status = "ok", version, host = Environment.MachineName }));

app.MapGet("/api/customers/{id}/bill", (string id) =>
{
    if (!customers.TryGetValue(id, out var c))
        return Results.NotFound(new { error = "unknown customer", id });

    var plan = plans[c.Plan];
    var extraGb = Math.Max(0, c.UsageGb - plan.IncludedGb);
    var charge = plan.MonthlyFee + (decimal)extraGb * plan.PricePerExtraGb;
    return Results.Ok(new
    {
        c.Id, c.Name, c.Country, c.Plan,
        usageGb = c.UsageGb,
        includedGb = plan.IncludedGb,
        extraGb,
        charge = Math.Round(charge, 2),
        currency = "EUR",
        servedBy = Environment.MachineName,
        version
    });
});

// Holds each request open so many requests overlap. This is what triggers an HTTP scale rule.
app.MapGet("/api/load", async (int? ms) =>
{
    var wait = Math.Clamp(ms ?? 2000, 100, 10000);
    await Task.Delay(wait);
    return Results.Ok(new { waitedMs = wait, servedBy = Environment.MachineName, version });
});

app.Run();

record Plan(string Name, decimal MonthlyFee, int IncludedGb, decimal PricePerExtraGb);
record Customer(string Id, string Name, string Plan, string Country, double UsageGb);
EOF

ls -la
wc -l Program.cs
```

**Expect:** `ls` lists `Dockerfile`, `.dockerignore`, `NovaTel.Billing.csproj` and `Program.cs`, and `wc -l` prints `60 Program.cs`. If the count differs, a paste was cut short: delete the file and paste that block again.

### Step 4: Know the test customers

The service holds `c-1001` to `c-1020`. Two you will use later:

| Customer | Plan | Usage | Expected charge |
| --- | --- | --- | --- |
| `c-1003` | Pro | 7.5 GB | 40.00 EUR |
| `c-1010` | Plus | 25 GB | 32.50 EUR (5 GB over the 20 GB included) |

The trainer ran this exact code on a laptop with .NET 8 and got these values. The container build itself has not been run yet in this doc, so that is Lab 1.

## Demo 1 and Lab 1: Build an image in Azure, no Docker

**Idea.** `az acr build` zips your folder, uploads it to your registry, builds the Dockerfile on a build agent inside Azure and stores the image there. Your machine only needs a browser.

### Trainer demo (10 min)

The trainer shares the screen in Cloud Shell and the portal.

1. Show the folder: `ls ~/novatel-billing`. Read the Dockerfile aloud: two stages, SDK to build, small runtime to run.
2. Run the build and talk through the output while it runs:

```bash
cd ~/novatel-billing
az acr build --registry $ACR --image $IMAGE:$TAG --file Dockerfile .
```

3. Point out in the output: the upload of the source context, `Step 1/...` lines for each Dockerfile instruction, the final push, and the closing line `Run ID: ... was successful after ...`.
4. In the portal open the registry, then **Services**, **Repositories**, click `novatel-billing`, and show the tag `v1` with its digest and size.
5. Say the full image name out loud: `<login server>/novatel-billing:v1`. Registry, repository and tag are the address.

### Follow along (10 min)

Run the same build in your own Cloud Shell. If your variables are gone, repeat the block in **Prepare**, Step 2.

```bash
cd ~/novatel-billing
az acr build --registry $ACR --image $IMAGE:$TAG --file Dockerfile .
```

**Expect:** after about 1 to 3 minutes the last lines say the run was successful. Then confirm from the command line:

```bash
az acr repository show-tags --name $ACR --repository $IMAGE -o table
az acr show --name $ACR --query loginServer -o tsv
```

The first command lists `v1`. The second prints your exact login server. Copy it into your notes.

### Your lab: version 2 and a rebuild (20 min)

**Task 1.** Change the default version in the code from `v1` to `v2`, using this one command:

```bash
cd ~/novatel-billing
sed -i 's/?? "v1"/?? "v2"/' Program.cs
grep -n 'APP_VERSION' Program.cs
```

**Task 2.** Build the image again with the tag `v2`. Do not overwrite `v1`.

**Task 3.** In the portal, open your registry and find both tags under **Repositories**, then `novatel-billing`. Write down the digest of each. Are they the same?

**Task 4.** In the portal find the history of your builds. Look under **Services** for **Tasks** and its **Runs** list, or ask the trainer if the label differs. Match each run to the tag you built and note the run duration.

**Task 5.** Answer in one sentence each, in your notes: Why is the image built with a multi-stage Dockerfile? Why is `v1` kept rather than overwritten?

| Check | You should see |
| --- | --- |
| `grep` output | the line contains `?? "v2"` |
| Tags in `show-tags` | `v1` and `v2` |
| Digests | different for `v1` and `v2` |
| Runs list | at least two successful runs |

**Hint for Task 2.** Use the same command as before but with `v2` as the tag, for example `--image $IMAGE:v2`.

**Answers for Task 5.** The SDK image is large and only needed to compile, so the final image copies only the published output onto the much smaller runtime image. Keeping `v1` lets you roll back to a known good image by changing the tag you deploy.

**Stretch.** Run `az acr repository show --name $ACR --image $IMAGE:v2 -o table` and read the size and last update time. Run `az acr task list-runs --registry $ACR -o table` and compare it with the portal.

### If the build is blocked

Some subscriptions do not allow ACR Tasks builds. If `az acr build` returns an error that tasks are not supported or not permitted, do not spend time on it. Import a public sample image instead. This still needs no Docker:

```bash
az acr import --name $ACR --source mcr.microsoft.com/dotnet/samples:aspnetapp --image $IMAGE:v1
az acr repository show-tags --name $ACR --repository $IMAGE -o table
```

You then continue with Demo 2 and Lab 2 using the sample image. It listens on port 8080 like the billing service but has no `/api` routes, so use its home page instead of `/health`. The trainer will tell you if this applies to the whole class.

**Key point to say aloud:** an image is a stack of read-only layers addressed by a digest. A tag such as `v1` is only a movable label on a digest.

## Demo 2 and Lab 2: Run the image on ACI and Container Apps

**Idea.** The same image runs in two places. ACI starts one container group quickly and bills while it runs. Container Apps adds a managed HTTPS address, revisions and scaling. You will deploy both from the portal.

### Part A: Azure Container Instances

**Trainer demo (8 min).** The trainer creates the instance live, then you repeat it.

1. In the portal click **Create a resource**, search **Container Instances**, click **Create**.
2. **Basics** tab:
   - **Resource group**: `<rg>`.
   - **Container name**: `aci-billing-<id>`.
   - **Region**: your `<region>`.
   - **Image source**: **Azure Container Registry**.
   - **Registry**: `acrnovatel<id>`. **Image**: `novatel-billing`. **Image tag**: `v1`.
   - **Size**: leave the default, about 1 vCPU and 1.5 GiB.
3. **Networking** tab: **Networking type** **Public**. **DNS name label**: `nvtl-billing-<id>`. **Ports**: `8080`, protocol **TCP**. Remove any default port 80 row.
4. **Advanced** tab: **Restart policy** **On failure**. Add the environment variable `APP_VERSION` = `aci`.
5. **Tags** tab: add `course`, `day`, `owner` as in Prepare.
6. **Review + create**, then **Create**. When complete, **Go to resource**.

**Expect:** **Status: Running**, and a **FQDN** such as `nvtl-billing-<id>.<region>.azurecontainer.io`. Copy it.

**Follow along (10 min).** Create your own instance. Then test in a browser tab:

- `http://<fqdn>:8080/health`
- `http://<fqdn>:8080/api/customers/c-1010/bill`

There is no `https`: ACI gives a plain HTTP endpoint on the port you opened.

**Expect:** the health reply contains `"version":"aci"`, and the bill for `c-1010` shows `"charge":32.50`.

Then in the instance menu open **Settings**, **Containers**, and look at the **Events**, **Properties**, **Logs** and **Connect** tabs. Read the event list: you should see the image being pulled, created and started.

**Pair task.** Ask your partner to open your URL. Does it respond? Now **Stop** your instance from the **Overview** toolbar. Refresh the partner's browser. What happens, and what do you still pay for?

| Check | You should see |
| --- | --- |
| Events tab | Pulling, Pulled, Created, Started |
| `servedBy` in the bill reply | `the container's own host name` |
| After Stop | the URL no longer answers; compute billing stops |

### Part B: Azure Container Apps

**Trainer demo (10 min).**

1. **Create a resource**, search **Container Apps**, **Create**.
2. **Basics** tab:
   - **Resource group**: `<rg>`. **Container app name**: `ca-billing-<id>`. **Region**: your `<region>`.
   - **Container Apps Environment**: **Create new**. Name it `cae-novatel-<id>`. Keep the plan **Consumption**. Leave the monitoring defaults. Add the three tags on the environment as well. Click **Create**.
3. **Container** tab:
   - Clear **Use quickstart image**.
   - **Image source**: **Azure Container Registry**. **Registry**: `acrnovatel<id>`. **Image**: `novatel-billing`. **Image tag**: `v1`.
   - **CPU and memory**: 0.25 cores and 0.5 Gi.
   - **Environment variables**: `APP_VERSION` = `aca-1`.
4. **Ingress** tab: **Enabled**. **Ingress traffic**: **Accepting traffic from anywhere**. **Ingress type**: **HTTP**. **Target port**: `8080`.
5. **Tags** tab: add the three tags. **Review + create**, then **Create**.
6. **Go to resource**. On **Overview** copy **Application Url**.

**Expect:** an `https://ca-billing-<id>.<random>.<region>.azurecontainerapps.io` address that answers with HTTPS.

**Follow along (15 min).** Create your own app, then browse to `<application url>/health` and `<application url>/api/customers/c-1003/bill`.

**Expect:** health shows `"version":"aca-1"` and the `c-1003` charge is `40.00`.

### Your lab: a second revision (12 min)

**Task 1.** Open your container app. Under **Application** open **Revisions and replicas**. Note the name of the first revision and its **Active** state.

**Task 2.** Click **Create new revision**. Change the image **Tag** from `v1` to `v2` and the environment variable `APP_VERSION` to `aca-2`. Create it.

**Task 3.** Refresh the revision list. How many revisions are active? Browse to `/health` again. Which version does it return?

**Task 4.** Switch back by activating the first revision and deactivating the second, or read the stretch below.

| Check | You should see |
| --- | --- |
| Revision list | two revisions, names ending in different suffixes |
| `/health` after the change | `"version":"aca-2"` while only the new revision is active |
| Replicas | at least 1 running replica for the active revision |

**Hint for Task 2.** The page for a new revision is the same form as the original container settings. Choose the existing container and click **Edit**.

**Stretch.** In **Revisions and replicas** switch **Revision mode** to **Multiple**, then in the revision list set traffic to 50 and 50 percent between the two revisions. Call `/health` ten times and count how many replies say `aca-1` and how many say `aca-2`.

**Say aloud:** a revision is an immutable snapshot of the container settings. A new revision means safe rollout and instant rollback, which a single App Service instance does not give you without slots.

## Lab 3: Scale under load, break it, and choose the service

Before you start, make sure only your `v1` or `v2` revision is active and `/health` answers over HTTPS. Set a variable in Cloud Shell:

```bash
URL=https://<your application url without a trailing slash>
curl -s $URL/health
```

### Part A: Add an HTTP scale rule

**Trainer demo (7 min).** Explain the idea first: with an HTTP rule, Container Apps adds a replica for every N concurrent requests, up to the maximum you allow.

1. Open your container app. Under **Application** open **Scale**.
2. Click **Edit and deploy**, then open the **Scale** tab.
3. Set **Min replicas** to `1` and **Max replicas** to `5`.
4. Click **Add**. Set **Rule name** `http-rule`, **Type** **HTTP scaling**, **Concurrent requests** `5`. Click **Add**.
5. Click **Create** to create the new revision. Wait until its state is **Running**.

**Follow along (8 min).** Do the same on your app.

**Expect:** the new revision lists `http-rule` under scale rules and shows 1 replica.

### Part B: Generate load and watch replicas (your lab, 20 min)

The `/api/load` endpoint holds each request for up to 10 seconds, so many requests overlap. You start the load from Cloud Shell with `curl`.

**Task 1.** Count the replicas before the load. In the portal open **Application**, **Revisions and replicas**, click the active revision, and open its **Replicas** tab. Write down the count.

**Task 2.** Start the load. This sends 30 requests at once, six times, about 10 seconds apart:

```bash
for round in 1 2 3 4 5 6; do
  for i in $(seq 1 30); do
    curl -s -o /dev/null "$URL/api/load?ms=10000" &
  done
  sleep 10
done
wait
echo load finished
```

**Task 3.** While it runs, refresh the **Replicas** tab every 30 seconds. Write down the maximum count you see and roughly how long the increase took.

**Task 4.** Straight after the load finishes, see which replicas answer:

```bash
for i in $(seq 1 20); do curl -s $URL/health; echo; done | sort | uniq -c
```

Each distinct `host` value is a different replica.

**Task 5.** Wait about 5 minutes and check the replica count again.

| Check | You should see |
| --- | --- |
| Before the load | 1 replica |
| During the load | 2 to 5 replicas, never more than 5 |
| `uniq -c` output | more than one host value while replicas are up |
| About 5 minutes after the load | back to 1 replica |

The increase is not instant, and scale-in waits for a cool-down period of several minutes. Your numbers will differ slightly from the neighbour's. That is normal.

**Hint for Task 3.** If you never get past 1 replica, check the rule says 5 concurrent requests and that the load command is still running. Run it again.

**Stretch.** Change **Min replicas** to `0`, wait until the app idles at zero replicas, then time the first request: `time curl -s $URL/health`. Compare it with a warm call. This is the cold start trade-off.

### Part C: Break cards (pair task, 20 min)

Work in pairs. Partner A drives the mouse in A's own app. Partner B sits at A's screen and may only say what to click.

A secretly picks one card and applies it to A's own container app as a new revision. B diagnoses from the portal: the revision state, **Log stream** and **System logs**, **Revisions and replicas**. Then swap. Never touch the other person's app from your own account.

| Card | A applies this to a new revision | What B should find |
| --- | --- | --- |
| 1 | Change the image tag to `v9` | The new revision fails to start. System logs mention the image or manifest was not found. |
| 2 | Change ingress **Target port** to `80` | The revision runs but the URL returns an error or times out. The app listens on 8080. |
| 3 | Turn **Admin user** off on the registry, then create a new revision | The revision cannot pull the image. System logs mention authorisation or credentials. |

After each diagnosis, A repairs the app by activating the previous good revision or creating a new correct one. If you turned admin user off for card 3, turn it back on.

The exact wording of the messages varies, so B looks for the cause, not for a specific sentence.

### Part D: Which service would you choose? (10 min)

In pairs, fill in the third column for each NovaTel workload. Use the comparison table in **Scenario and outcomes** and what you saw today.

| Workload | Needs | Your choice (ACI, ACA or App Service) and one reason |
| --- | --- | --- |
| Nightly rebuild of invoices, runs 20 minutes, no users | Runs once, then goes away |  |
| Billing API with month-end peaks, new versions weekly | Scaling, safe rollout |  |
| Customer self-care site with custom domain and staging slot | Always on, classic web app features |  |

**Suggested answers.** Nightly rebuild: ACI, a short job billed by the second (or a Container Apps job). Billing API: ACA, because of HTTP scaling and revisions. Self-care site: App Service, for custom domains, slots and a predictable plan. There are valid alternatives if the reason is sound.

## Quiz, cleanup and troubleshooting

### Quiz (10 min, closed book)

Answer on paper first, then check below. The reason matters more than the letter.

**Q1.** The sandbox has no Docker. Why does `az acr build` still work?

- A. Cloud Shell contains a hidden Docker engine that your laptop uses
- B. The upload is built by a build agent inside Azure Container Registry
- C. The registry converts source code to an image without a Dockerfile
- D. It does not work without Docker

**Q2.** In `acrnovatelnv07.azurecr.io/novatel-billing:v2`, which part can be moved to point at a different image later?

- A. The registry name
- B. The repository name
- C. The tag `v2`
- D. The digest

**Q3.** Which service gives you a rule such as "add a replica for every 5 concurrent requests" without extra tooling?

- A. Azure Container Instances
- B. Azure Container Apps
- C. Azure Container Registry
- D. Cloud Shell

**Q4.** Your app has min 1, max 5 replicas and an HTTP rule of 5 concurrent requests. About 23 requests are in flight at once. Roughly how many replicas do you expect?

- A. 1
- B. 3
- C. 5
- D. 23

**Q5.** In single revision mode you deploy a new revision with a missing image tag. What happens to users?

- A. Everyone gets errors immediately
- B. The previous healthy revision keeps serving while the new one stays failed
- C. The registry deletes the old image
- D. The app scales to zero

**Q6.** Why is the registry **Admin user** acceptable in this lab but not in production?

- A. It is slower
- B. It is a shared password with full registry rights. Production uses a managed identity with the `AcrPull` role.
- C. It disables TLS
- D. It cannot pull images

**Answers**

| Q | Answer | Reason |
| --- | --- | --- |
| 1 | B | ACR Tasks runs the Dockerfile build on Azure build agents and stores the result in your registry. |
| 2 | C | A tag is a movable label. The digest identifies the exact content and does not move. |
| 3 | B | Container Apps has built-in HTTP, queue and custom scale rules. ACI has none. |
| 4 | C | 23 requests at 5 per replica is about 5 replicas (rounded up), and the maximum is 5. |
| 5 | B | Container Apps does not move traffic to a revision that is not healthy. Check the revision state and system logs. |
| 6 | B | A managed identity needs no stored password and can be limited to pulling images. |

**Score.** 6 of 6: ready to discuss container hosting choices with a project team. 4 or 5: redo Lab 3 Part C. Below 4: repeat Lab 1 and Lab 2 Part B, then retake the quiz.

### Clean up at the end of the day

Container Apps and Container Instances cost money while they run, so do not skip this. Everything you made sits in your own resource group.

**Step 1: Stop what is running.** Open `aci-billing-<id>` and click **Stop** if it is still running.

**Step 2: Check the cleanup list.** Run the preview first (trainer or anyone with the Day 2 scripts), from the folder that holds the scripts:

```powershell
.\scripts\cleanup-day2.ps1 -Id <id> -ResourceGroup rg-nvtl-oct26-<id> -Owner <id> -Preview
```

The list must include your container registry, your container instance, your container app and its environment. It must not include anyone else's resources.

**Step 3: Check the extras.** The Container Apps environment can create a Log Analytics workspace that may not carry your tags, so the tag-based script can miss it. Open your resource group, sort by type, and look for a **Log Analytics workspace** you did not create on purpose. If it has no `owner = <id>` tag, add the tag now or delete it yourself.

**Step 4: Delete.**

```powershell
.\scripts\cleanup-day2.ps1 -Id <id> -ResourceGroup rg-nvtl-oct26-<id> -Owner <id>
```

No script? In the portal, open `<rg>`, tick only the resources that carry your `owner` tag, and click **Delete**. Delete the container app first, then its environment.

**Expect:** your resource group no longer lists the registry, the instance, the app or the environment. The resource group itself stays.

**On your side.** Close Cloud Shell. If it asked you to create a storage account, tell the trainer. Delete any copied URLs from your notes if the trainer asks.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Registry name rejected | Needs 5 to 50 lowercase letters and digits, and must be unique | Remove hyphens, add a digit |
| Cannot see my resource group | Wrong account or not created yet | Ask the trainer. Do not create another one. |
| Cloud Shell will not start or asks for storage | Subscription policy or first use | Use the no-storage option if shown, otherwise ask the trainer |
| `ID`, `RG` or `ACR` shows empty | Cloud Shell restarted and lost its variables | Repeat the variables block in Prepare, Step 2 |
| `az acr build` says tasks are not supported or not permitted | Subscription does not allow ACR Tasks | Use `az acr import` as shown under Lab 1 |
| `az acr build` fails reading the source folder | The folder or Dockerfile is missing | Run `ls ~/novatel-billing`. Repeat Prepare, Step 3. |
| Build fails at `dotnet restore` | Package source not reachable from the build agent | Retry once. If it persists, tell the trainer and use the import fallback. |
| ACI create fails on the image | Admin user not enabled, or wrong tag | Check **Access keys** on the registry and the tag in **Repositories** |
| ACI URL times out | Wrong port or you used `https` | Use `http://<fqdn>:8080` |
| Container app URL shows an error page | Target port is not 8080, or the revision failed | Check **Ingress** and the revision state |
| No extra replicas appear | Rule not saved, or load too light | Check the rule on the active revision and run the load again |
| Replicas stay high after the load | Scale-in has a cool-down | Wait about 5 minutes |
| Quota or core limit error | Shared subscription limits | Tell the trainer, use a smaller size, or pair with a neighbour |

### What has and has not been checked

| Item | Status |
| --- | --- |
| The .NET billing service: builds and runs, `/health`, bills for `c-1003` (40.00) and `c-1010` (32.50), 404 for an unknown customer, `/api/load` waits the requested time | Run by me on .NET 8 outside Docker. The code and values are real. |
| `az acr build` and `az acr import` need no local Docker; ACR Tasks quick tasks; Cloud Shell cannot run `docker run` | Checked against Microsoft Learn |
| Registry login server can include a hash suffix | Checked against Microsoft Learn |
| The Dockerfile, the container build in ACR, and the image running on ACI and ACA | **Not run by me.** The Dockerfile follows the standard multi-stage .NET pattern. Verify in the dry run. |
| Exact portal labels and positions (Container Apps **Scale**, **Revisions and replicas**, **Create new revision**, ACI **Containers** tabs, ACR **Tasks** and **Runs**) | **Not run in a live portal.** Check in the dry run and fix the wording here. |
| Whether the portal uses the admin credentials automatically when the source is your registry | Not confirmed. If it asks for credentials, use the user name and password under **Access keys**. |
| Scale timing and the break-card message wording | Varies. Observed numbers must replace the expectations after the dry run. |
| Whether Cloud Shell in this subscription offers the no-storage option and allows `az acr build` from a local folder | Not confirmed. A Microsoft tutorial uses a Git URL for the build context in Cloud Shell. If the local folder fails, tell the trainer. |

### Trainer dry run (45 min, the day before)

1. In the trainer resource group with ID `nvtrain`, do Prepare, including the four source files, and check `wc -l Program.cs` gives 60.
2. Run `az acr build` and note the real duration. Record it in the timing table.
3. Create the ACI and ACA exactly as written. Fix any label that differs and note whether credentials were asked for.
4. Add the scale rule, run the load, and replace the expected replica counts and timings with what you see.
5. Run all three break cards and write down the real messages.
6. Run the cleanup preview and confirm the Log Analytics workspace is listed or deleted by hand.
7. Tell the class in advance what to do if ACR Tasks is blocked in the subscription.
