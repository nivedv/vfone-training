# Day 2 Session 2: Blob Storage Demo and Lab (Portal)

Oct 14, 2026 · @Nived Varma

This is one doc for the trainer demo, the in-class lab and your own practice later. Everything is done in the Azure portal in your browser; nothing needs Docker or Visual Studio. In class it takes about two hours: two demos, three labs and a short quiz, after a 15 minute Prepare.

## How to use this doc

**Who it is for.** The trainer runs the demos from it. Participants follow along during the demo, do the labs in class, and repeat them alone afterwards. Every step says where to click and what you should see.

**The pattern in each part**

1. **Trainer demo.** The trainer does the task on screen and explains why. Steps marked **Follow along** are ones you may repeat at the same time. Steps marked **Watch** are for the trainer only.
2. **Your lab.** You repeat the same task with your own names and then do extra tasks. Tick each task box (`- [ ]`) as you finish it.
3. **Check.** Each lab ends with a short list of things you must be able to see. If one is missing, go to Troubleshooting at the end.
4. **Answer.** Write the answers to the questions in your own notes. Compare them with the quiz at the end.

**Labels used throughout**

| Label | Meaning |
| --- | --- |
| **Trainer demo** | The trainer drives. You watch, or follow along where told. |
| **Follow along** | Do this step yourself while the trainer does it. |
| **Your lab** | You work alone or in your pair. |
| **Pair task** | Two people work together and swap roles halfway. |
| **Expect** | What you should see on screen when the step worked. |
| **Stretch** | Optional, for when you finish early. |
| **Bold words** | The exact label on a button, blade or field in the portal. |

**Fill in your own values once.** Replace `<id>` everywhere in this doc with your own ID. Your trainer gives you the ID and your resource group.

| Item | Your value |
| --- | --- |
| Your ID (lowercase letters and digits, 2 to 8 characters) |  |
| Your resource group | `rg-nvtl-oct26-<id>` unless your trainer says otherwise |
| Your storage account (created in Prepare) | `stnovatel<id>` |
| Your region | Same as your resource group |

**Rules for the shared subscription**

- Work **only** in your own resource group. Never open, change or delete another person's resources.
- Add the tags in Prepare (`course=azure-dev`, `day=2`, `owner=<id>`) to everything you create. The end-of-day cleanup finds your resources by those tags.
- Names of storage accounts are global, so they must be unique: always include your ID.
- Treat account keys and SAS links as passwords. Do not paste them in chat or email.

**If the portal looks different.** Azure changes menus from time to time. Look for the **bold** blade or button name rather than its position. If a name has moved, use the search box at the top of the portal and tell the trainer.

**Practising later.** Make a copy of this doc, replace `<id>` with your ID, and work through it again. Resources from class are deleted at the end of the day, so a later practice run starts from Prepare.

**What you hand in (if the trainer asks).** Your answers to the questions and a screenshot of each **Check** list.

## Scenario and outcomes

**The story.** NovaTel issues a monthly invoice to every customer in India, the UK and Germany. The invoices must be stored cheaply, found by customer and month, kept for the legal retention period and then deleted. A debt-collection partner must be able to read **one** invoice without seeing any other. You build all of that in Azure Blob Storage, using only the portal.

**By the end you can**

1. Create a storage account and containers and say why NovaTel uses one container per market.
2. Upload invoice documents and set their **properties** (content type), **metadata** (customer, month, country) and **index tags** (searchable).
3. Move a blob between the Hot, Cool, Cold and Archive tiers and explain the cost and access trade-off.
4. Write a **lifecycle management** rule that cools, archives and deletes invoices by age, and read the rule as JSON.
5. Generate a **shared access signature (SAS)** with the right permissions, time window and protocol, and work out from the link why a broken one fails.
6. Use the storage firewall (network rules) and explain when it blocks a caller who holds a valid SAS.

**Timing in class**

| Part | What | Minutes |
| --- | --- | --- |
| Prepare | Account, containers, synthetic invoices | 15, before class or at the start |
| Demo 1 | Upload and describe invoices | 15 |
| Lab 1 | Upload, metadata, tags, tiers across three containers | 40 |
| Demo 2 | Lifecycle management policy | 15 |
| Lab 2 | Retention rule for each market | 20 |
| Lab 3 | Break and fix a SAS link, then network rules | 20 |
| Quiz | Closed-book check | 10 |

**The retention rule NovaTel follows in this lab**

| Invoice age | What happens | Why |
| --- | --- | --- |
| 0 to 30 days | Stays in **Hot** | Customers and agents open recent invoices often |
| After 30 days | Moves to **Cool** | Cheaper to store, a little dearer to read |
| After 90 days | Moves to **Archive** | Cheapest, but offline: reading takes hours |
| After 7 years (2,555 days) for India and the UK, 10 years (3,650 days) for Germany | **Deleted** | Legal retention ends |

These numbers are teaching values chosen for this course, not legal advice.

## Prepare: your storage account and your invoices (15 min)

Every participant, and the trainer, does this once. It is **not** a demo: do it yourself in your own resource group.

### Step 1. Sign in and find your resource group

1. Open `https://portal.azure.com` in your browser and sign in with the lab account your trainer gave you.
2. In the search box at the top type **Resource groups** and open it.
3. Click your resource group, `rg-nvtl-oct26-<id>`.

**Expect:** the resource group opens and shows **0 resources** (or only the items your trainer put there). If you cannot see it, tell the trainer: do not create a different one.

### Step 2. Create the storage account

1. Inside your resource group click **+ Create**, search for **Storage account**, then **Create**.
2. On the **Basics** tab set:

| Field | Value |
| --- | --- |
| Subscription | The shared lab subscription (already selected) |
| Resource group | Your resource group |
| Storage account name | `stnovatel<id>` (3 to 24 characters, lowercase letters and digits only) |
| Region | The same region as your resource group |
| Preferred storage type | **Azure Blob Storage or Azure Data Lake Storage Gen 2** |
| Performance | **Standard** |
| Redundancy | **Locally-redundant storage (LRS)** |

3. Click **Next** to **Advanced** and check: **Enable hierarchical namespace** is **off**, **Allow enabling anonymous access on individual containers** is **off**, **Minimum TLS version** is **Version 1.2**, **Access tier** is **Hot**.
4. Click **Next** until **Networking** and check **Public network access** is **Enabled from all networks**. You change this on purpose in Lab 3.
5. Leave **Data protection** and **Encryption** at their defaults.
6. On the **Tags** tab add three tags exactly:

| Name | Value |
| --- | --- |
| `course` | `azure-dev` |
| `day` | `2` |
| `owner` | `<id>` |

7. Click **Review + create**, then **Create**. Wait for **Your deployment is complete**, then click **Go to resource**.

**Expect:** the storage account overview page shows your account name, your resource group and **Performance: Standard**, **Access tier (default): Hot**.

If the name is taken or rejected, add a digit to your ID part (for example `stnovatel<id>1`) and use that name everywhere in the rest of this doc.

### Step 3. Create the containers

In your storage account, in the left menu under **Data storage**, click **Containers**. For each name below click **+ Container**, type the name, leave **Anonymous access level** on **Private (no anonymous access)**, then **Create**.

| Container | Purpose |
| --- | --- |
| `invoices-in` | India invoices |
| `invoices-uk` | UK invoices |
| `invoices-de` | German invoices |
| `lifecycle-demo` | Practice container for the lifecycle demo |

**Expect:** four containers in the list (plus `$logs` if shown).

### Step 4. Generate synthetic invoices on your Windows machine

The invoices are **made-up** documents with invented names and amounts. No real customer data is used anywhere in this course.

1. Open **Windows PowerShell** (not as administrator).
2. Paste the whole script below and press Enter. It only creates local files; it does not contact Azure.

```powershell
$root = "C:\novatel\invoices"
$markets = @(
  @{ Code = "IN"; Folder = "in"; Currency = "INR"; TaxName = "GST"; TaxRate = 0.18; Names = @("Asha Rao", "Vikram Shah", "Meera Nair", "Rohan Iyer", "Priya Das") },
  @{ Code = "UK"; Folder = "uk"; Currency = "GBP"; TaxName = "VAT"; TaxRate = 0.20; Names = @("Oliver Hughes", "Amelia Clarke", "Harry Patel", "Isla Reid", "Jack Morgan") },
  @{ Code = "DE"; Folder = "de"; Currency = "EUR"; TaxName = "MwSt"; TaxRate = 0.19; Names = @("Jonas Weber", "Lena Fischer", "Felix Wagner", "Mia Becker", "Lukas Hoffmann") }
)
$plans = @("Prepaid-Basic", "Postpaid-Standard", "Postpaid-Premium")
$fees  = @(9, 49, 79)
$rnd = New-Object System.Random 42

foreach ($m in $markets) {
  $dir = Join-Path $root $m.Folder
  New-Item -ItemType Directory -Path $dir -Force | Out-Null
  for ($i = 1; $i -le 10; $i++) {
    $n = "{0:0000}" -f $i
    $name = $m.Names[($i - 1) % $m.Names.Count]
    $cust = 1000 + $rnd.Next(1, 200)
    $p = $rnd.Next(0, 3)
    $data = $rnd.Next(1, 40)
    $net = [math]::Round($fees[$p] + $data * 0.5, 2)
    $tax = [math]::Round($net * $m.TaxRate, 2)
    $total = [math]::Round($net + $tax, 2)
    $lines = @(
      "NOVATEL TELECOM - INVOICE",
      "Invoice number : INV-$($m.Code)-$n",
      "Invoice month  : 2026-09",
      "Customer ID    : c-$cust",
      "Customer name  : $name",
      "Country        : $($m.Code)",
      "Plan           : $($plans[$p]) ($($m.Currency) $($fees[$p]))",
      "Data used      : $data GB at 0.50 per GB",
      "Net amount     : $($m.Currency) $net",
      "$($m.TaxName)             : $($m.Currency) $tax",
      "TOTAL DUE      : $($m.Currency) $total",
      "SYNTHETIC DOCUMENT FOR TRAINING - NOT A REAL INVOICE"
    )
    Set-Content -Path (Join-Path $dir "INV-$($m.Code)-$n.txt") -Value $lines -Encoding ASCII
  }
}
Get-ChildItem $root -Recurse -File | Group-Object DirectoryName | Select-Object Count, Name
```

**Expect:** the last line prints three rows, each with **Count 10**, for `C:\novatel\invoices\in`, `...\uk` and `...\de`. Open `C:\novatel\invoices\uk\INV-UK-0001.txt` in Notepad: it should read like a short invoice that ends with the line `SYNTHETIC DOCUMENT FOR TRAINING - NOT A REAL INVOICE`.

The file names `INV-IN-0001.txt` to `INV-IN-0010.txt`, `INV-UK-...` and `INV-DE-...` are used in every step below.

### Check before you continue

- [ ] The storage account `stnovatel<id>` exists in my resource group with the three tags
- [ ] Four containers exist, all **Private**
- [ ] `C:\novatel\invoices` has three folders with 10 files each

The trainer does the same preparation in the trainer resource group before class, with ID `nvtrain`.

## Demo 1 and Lab 1: Upload and describe invoices (15 + 40 min)

**What you learn.** A blob is more than a file. It has **properties** (size, content type, tier), **metadata** (your own name and value pairs, returned with the blob) and **index tags** (name and value pairs you can search by). You will set all three and move a blob between tiers.

### Trainer demo: the UK invoices (15 min)

The trainer does this in their own account. **Follow along** steps may be repeated by participants at the same time in their own account (the UK container).

| # | Step | Where to click | Expect | Say |
| --- | --- | --- | --- | --- |
| 1 | **Follow along.** Upload the 10 UK invoices | Storage account, **Containers**, `invoices-uk`, **Upload**. **Browse for files**, select the 10 files in `C:\novatel\invoices\uk`. Open **Advanced**, set **Blob type** to **Block blob**, **Upload to folder** to `2026/09`. Click **Upload** | 10 files listed as uploaded, then a folder `2026` in the container | A container is a flat list of names. `2026/09/INV-UK-0001.txt` is one name with slashes in it; the "folders" are only a prefix the portal shows. |
| 2 | **Follow along.** Open one blob | Click `2026`, `09`, then `INV-UK-0001.txt` | An overview with **URL**, **Last modified**, **Size**, **Access tier: Hot**, **Type: Block blob**, and a **Content-Type** such as `text/plain` | These are the **properties**. The portal set the content type from the `.txt` extension. A browser uses it to decide whether to show or download the file. |
| 3 | **Follow along.** Add metadata | On the blob overview find **Metadata** (scroll down), click **Edit**, then add four rows: `customerid` = the customer ID in the file (for example `c-1042`), `invoicemonth` = `2026-09`, `country` = `UK`, `status` = `open`. **Save** | The four rows are listed under **Metadata** | **Metadata** travels with the blob and costs nothing to read with it, but you cannot search by it. Names are simple words, values are plain ASCII text. |
| 4 | **Follow along.** Add index tags | On the same blob open **Blob index tags**, add `country` = `UK` and `status` = `open`, **Save** | Two tags listed | **Index tags** are what you can **search**. Use tags for what you will ask about (status, country) and metadata for descriptive extras. |
| 5 | **Follow along.** Search by tag | Go back to the container list of blobs. In the container's list of blobs click the **Blob index tags** filter button, then enter key and value `status` = `open` | Only the tagged blob is listed | The service answers this without opening every blob. Metadata could not do this. |
| 6 | **Follow along.** Change the tier | Open the blob, click **Change tier**, choose **Cool**, **Save** | **Access tier: Cool** and a new **Access tier last modified** time | Cool costs less to store and a little more to read, with a 30 day minimum stay. |
| 7 | **Watch.** Archive and try to read | On a second blob, **Change tier** to **Archive**, then click **Download** | The blob shows **Archive**; download is refused with a message that the blob is archived | Archive is **offline**. To read it you must rehydrate it back to Hot or Cool, which takes hours. Plan Archive for data nobody expects to open. |
| 8 | **Follow along.** Download and open | Open a **Hot** blob, **Download**, open it in Notepad | The invoice text | A blob in Hot or Cool reads immediately. |
| 9 | **Predict, then watch.** Overwrite | Upload `INV-UK-0001.txt` again to `2026/09`. First leave **Overwrite if files already exist** off, then repeat with it on | Off: an error that the blob exists. On: replaced | Ask first: "Will the metadata survive?" Then open the blob: a re-upload replaces the whole blob, so check what happened to **Metadata** and the tier and explain it. |

The trainer closes the demo with the three sentences:

1. "The **name** of a blob is its path."
2. "**Metadata** describes, **tags** find."
3. "**Archive** is cheap but offline."

### Your lab: the India and German invoices (40 min)

Work alone. Use your own storage account. Tick each box as you finish.

**Task 1. Upload (10 min)**

- [ ] Upload the 10 files from `C:\novatel\invoices\in` to container `invoices-in`, folder `2026/09`
- [ ] Upload the 10 files from `C:\novatel\invoices\de` to container `invoices-de`, folder `2026/09`
- [ ] If you did not follow along in the demo, also upload the 10 UK files to `invoices-uk`, folder `2026/09`

**Expect:** each container has a folder `2026`, then `09`, with 10 blobs.

**Task 2. Metadata (10 min)**

Open the first blob of each market (`INV-IN-0001.txt`, `INV-UK-0001.txt`, `INV-DE-0001.txt`). In each, read the **Customer ID** from the file text (download it, or open the blob's **Edit** tab to read it) and set four metadata rows:

| Name | Value |
| --- | --- |
| `customerid` | The ID from that file, for example `c-1042` |
| `invoicemonth` | `2026-09` |
| `country` | `IN`, `UK` or `DE` to match |
| `status` | `open` |

- [ ] Metadata set on `INV-IN-0001.txt`
- [ ] Metadata set on `INV-UK-0001.txt`
- [ ] Metadata set on `INV-DE-0001.txt`
- [ ] Change `status` to `paid` on `INV-DE-0001.txt` **without losing the other three rows**

**Expect:** each blob lists four metadata rows, and the German one shows `status = paid`.

**Task 3. Index tags and search (8 min)**

- [ ] On the same three blobs add tags `country` and `status` (same values as the metadata)
- [ ] In `invoices-de`, search by tag `status` = `paid`: exactly one blob is found
- [ ] In `invoices-uk`, search by tag `status` = `open`: one blob is found

**Task 4. Tiers (7 min)**

- [ ] Move `INV-IN-0001.txt` to **Cool** and check the tier changed
- [ ] Move `INV-IN-0002.txt` to **Cold** (if the option is offered) and note any message about minimum storage days
- [ ] Move `INV-IN-0010.txt` to **Archive**, then try to download it and write down the exact message
- [ ] Do **not** wait for rehydration: setting an archived blob back to Hot starts a rehydration that takes hours

**Task 5. Overwrite (5 min)**

- [ ] Predict: if you re-upload `INV-IN-0001.txt` with **Overwrite** on, will its metadata and Cool tier be kept?
- [ ] Test it, then open the blob and note what happened to the metadata, the tags and the tier

**Check**

| What | Expected |
| --- | --- |
| Blobs per container under `2026/09` | 10 each |
| `INV-DE-0001.txt` | 4 metadata rows, `status = paid`, 2 tags |
| Tag search `status = paid` in `invoices-de` | 1 result |
| Archived blob | Download refused, tier shows **Archive** |

**Good to know.** Setting metadata rewrites the blob's last-modified time; setting tags does not. A changed last-modified time restarts the blob's age for lifecycle rules, and metadata cannot be changed on an archived blob. Metadata names must follow C# identifier rules, so use simple words such as `customerid`.

**Answer in your notes**

1. A partner needs all invoices for customer `c-1042` in September. Would you search by metadata or by tag? Why?
2. Which tier would you pick for an invoice that is 2 years old and rarely opened? What is the catch?
3. What happens to a blob's metadata when you upload a new file with the same name?
4. Why can the portal show a "folder" that does not exist?

**Stretch**

1. Open **Storage browser** in the left menu and find the same blobs there.
2. Under the container **Properties** or **Access policy**, find and note the container's public access level.
3. In the storage account open **Data protection** and find **soft delete for blobs**. Delete one blob, then restore it.

## Demo 2 and Lab 2: A retention policy for invoices (15 + 20 min)

**What you learn.** A **lifecycle management policy** moves blobs to cheaper tiers and deletes them as they age, with no code. The rules are JSON, but you build and read them in the portal. The service evaluates the policy about **once a day**, so the effect is never instant.

**Key facts to remember**

- A storage account has **one policy** that holds **many rules**. The portal adds a rule to the policy when you save.
- A rule selects blobs by **type**, by **prefix** and optionally by **index tag**.
- The prefix **starts with the container name**: `invoices-uk/2026/` and not `2026/`.
- Age is measured from **last modified** (or optionally last access). Re-uploading a blob, or changing its metadata, resets its age.
- If a **delete** and a tier move both apply, **delete wins**.
- A new or changed policy can take up to **24 hours** to start acting, and each action then runs on the daily evaluation.

### Trainer preparation: the pre-baked result (do this two days before class)

You cannot show a day-old effect live, so prepare it. In the trainer account:

1. Container `lifecycle-demo`: **Upload** 3 files from `C:\novatel\invoices\uk` into the folder `aged` (so the names start `aged/`).
2. **Lifecycle management**, **+ Add a rule**, name `demo-tier-aged`, scope **Limit blobs with filters**, **Block blobs** and **Base blobs**. On **Base blobs** set: if last modified **more than 1 day ago**, **Move to cool storage**. On **Filter set** set **Blob prefix** `lifecycle-demo/aged/`. **Add**.
3. On the class day, open the three blobs: each shows **Access tier: Cool** and a recent **Access tier last modified**.

If the blobs are still Hot, the daily run has not reached them yet. Say so honestly and show the rule and the JSON instead.

### Trainer demo (15 min)

| # | Step | Where to click | Expect | Say |
| --- | --- | --- | --- | --- |
| 1 | **Watch.** Show the result first | Container `lifecycle-demo`, folder `aged`, open a blob | **Access tier: Cool** and **Access tier last modified** | "Nobody moved these blobs. The policy did, overnight." |
| 2 | **Watch.** Read the rule | Storage account, **Data management**, **Lifecycle management**, click the rule `demo-tier-aged`, then **Code view** | JSON with `prefixMatch` `lifecycle-demo/aged/` and `tierToCool` with `daysAfterModificationGreaterThan: 1` | The wizard and the JSON are the same thing. Point out `blobTypes`, `prefixMatch`, `baseBlob`. |
| 3 | **Follow along.** Add the UK and India retention rule | **+ Add a rule**. **Details:** name `retention-in-uk`, scope **Limit blobs with filters**, **Block blobs**, **Base blobs**, **Next**. **Base blobs:** enable three conditions: more than **30** days, **Move to cool storage**; more than **90** days, **Move to archive storage**; more than **2555** days, **Delete the blob**. **Next**. **Filter set:** **Blob prefix** `invoices-in/` and `invoices-uk/` (add each). **Add** | The rule appears in the list, status **Enabled** | Cool at 30, archive at 90, delete at 7 years. Two prefixes, one rule. |
| 4 | **Predict.** What changes tomorrow? | Open **Code view** of the new rule | The three actions in JSON | Ask: "Which September invoices will be tiered tomorrow?" None: they are days old, not 30. A rule is a promise about the future. |
| 5 | **Watch.** Conflicts and order | Explain with the rule JSON | None | If delete and tier would both apply to a very old blob, delete wins. Actions fire as the blob reaches each age. |
| 6 | **Watch.** Where the portal stops | Point at **Enable access tracking** in the lifecycle page | The option exists | Rules can also use last access time instead of last modified, if access tracking is on. |

Trainer closes with: "Policies are **prefix-driven**, **day-granular** and **once-a-day**. Design retention with those three words in mind."

### Your lab (20 min)

**Task 1. Practice rule for tomorrow**

- [ ] Container `lifecycle-demo`: upload 3 files from `C:\novatel\invoices\uk` into folder `aged`
- [ ] Add a rule `demo-tier-aged` exactly as in the trainer preparation (prefix `lifecycle-demo/aged/`, Cool after 1 day)
- [ ] Tomorrow, if you are practising again, open one of those blobs and write down its tier and **Access tier last modified**

**Task 2. The retention rules for NovaTel**

Build the two rules below in the portal with the wizard. Do not paste JSON yet.

| Rule name | Prefixes | Cool after | Archive after | Delete after |
| --- | --- | --- | --- | --- |
| `retention-in-uk` | `invoices-in/`, `invoices-uk/` | 30 days | 90 days | 2555 days |
| `retention-de` | `invoices-de/` | 30 days | 90 days | 3650 days |

- [ ] Rule `retention-in-uk` is in the list and **Enabled**
- [ ] Rule `retention-de` is in the list and **Enabled**
- [ ] Open **Code view** of each and read the JSON: find `prefixMatch` and the three `daysAfterModificationGreaterThan` values

**Expect:** three rules in the list: `demo-tier-aged`, `retention-in-uk`, `retention-de`.

**Task 3. Check your JSON against the reference**

Only after Task 2, compare your **Code view** of `retention-in-uk` with this reference. The order of properties may differ.

```json
{
  "enabled": true,
  "name": "retention-in-uk",
  "type": "Lifecycle",
  "definition": {
    "filters": {
      "blobTypes": ["blockBlob"],
      "prefixMatch": ["invoices-in/", "invoices-uk/"]
    },
    "actions": {
      "baseBlob": {
        "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
        "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
        "delete":        { "daysAfterModificationGreaterThan": 2555 }
      }
    }
  }
}
```

- [ ] My JSON matches the reference (same prefixes and numbers)

**Task 4. Think about it**

- [ ] Disable `retention-de` with its toggle or **Edit**, then enable it again. Note what the list shows in between.

**Answer in your notes**

1. Which tier will a 100-day-old UK invoice be in once the policy has run? A 100-day-old German one?
2. Will tomorrow's run change any of today's September invoices? Why not?
3. Why must the prefix start with the container name?
4. What happens to a blob's age if someone overwrites it?
5. Germany keeps invoices for 10 years and the UK for 7. One rule or two? Why?

**Stretch**

1. Add a rule that deletes blobs whose **index tag** `status` equals `void` after 1 day, using the **Blob index match** option on the **Filter set** tab.
2. Find the **Enable access tracking** option and explain when you would use last access instead of last modified.
3. In the portal, **Overview** of the storage account, open the **Monitoring** or **Insights** area and see whether any lifecycle activity is shown yet.

## Lab 3: Why can't the partner read the invoice? (20 min, in pairs)

**Scenario.** A debt-collection partner was sent a link to read **one** invoice and gets an error. Your job is to find out why from the link and the error text alone, give them a working link and keep the account private. In this lab each person breaks a link on purpose for the other to diagnose.

**What a SAS link contains.** A **shared access signature (SAS)** is a URL that carries its own permission slip. The service checks each field and the signature. Read the fields from the part after the `?`:

| Field | Meaning | What to check |
| --- | --- | --- |
| `sv` | Service version | A date; not a problem by itself |
| `st` | Start time (UTC) | Is it in the future? |
| `se` | Expiry time (UTC) | Is it in the past? |
| `sr` | Resource: `b` is one blob, `c` is a whole container | Is the scope what the partner needs? |
| `sp` | Permissions: `r` read, `w` write, `d` delete, `l` list, `c` create | Does it include `r`? |
| `sip` | Allowed caller IP address or range | Is the caller's IP inside it? |
| `spr` | Allowed protocol: `https` or `https,http` | Is the caller using HTTPS? |
| `sig` | Signature over all of the above | Never edit it: any change breaks it |

The signature is made with the storage account key. That is why **editing a link never works**: you must create a new one.

### Trainer mini-demo (5 min): a good link

| # | Step | Where to click | Expect |
| --- | --- | --- | --- |
| 1 | Open the blob | `invoices-uk`, `2026`, `09`, `INV-UK-0001.txt` | Blob overview |
| 2 | Generate a SAS | **Generate SAS** tab. **Signing method:** **Account key**. **Permissions:** **Read** only. **Start** left at the portal default (or 15 minutes earlier), **Expiry** 15 minutes from now. **Allowed protocols:** **HTTPS only**. Leave **Allowed IP addresses** empty. Click **Generate SAS token and URL** | A **Blob SAS token** and a **Blob SAS URL** appear |
| 3 | Test it | Paste the **Blob SAS URL** into a **private/incognito** browser window | The invoice text appears |
| 4 | Decode it | Point at `sp=r`, `se=`, `sr=b`, `spr=https`, `sig=` in the URL | Matches the table above |

Say: "This link opens exactly one blob, for 15 minutes, read only, over HTTPS. Nothing else in the account is exposed." Copy the URL for the pair task only; do not share it outside the class.

### Pair task: break it and fix it

Work in pairs. **Person A** breaks a link; **Person B** diagnoses it. Then swap with the next card.

**Step 1. A creates the broken link.** In the **Generate SAS** tab of `INV-UK-0001.txt`, set everything as in the trainer mini-demo (Account key, Read, HTTPS only, 15 minutes, start time left at the default or set 15 minutes earlier), **except** for the one change on the card. Give B the **Blob SAS URL** only. **Do not tell B what the defect is.**

| Card | A changes | A also does |
| --- | --- | --- |
| 1. Too short | **Expiry** 2 minutes from now | Waits until after the expiry before B tests |
| 2. Wrong permission | Untick **Read**, tick **Write** | Nothing |
| 3. Wrong caller | **Allowed IP addresses** `203.0.113.1` | Nothing |
| 4. Wrong blob | Normal link for `INV-UK-0001.txt` | Tells B to try `INV-UK-0002.txt` by changing the file name in the URL path |
| 5. Edited link | Normal link | Tells B to change one character of the `se` value in the URL before testing |

**Step 2. B reproduces the error.** Paste the URL into an **incognito** browser window. Write down the **HTTP status** and the **error code and message** shown in the page (the page is XML: look for `<Code>` and `<Message>`). Do not stop at "403".

**Step 3. B decodes the link.** Copy the URL into Notepad and read each field using the table. Write one line: which field explains the error.

**Step 4. B proposes the fix.** In one sentence say what A must change when creating a new link. **Do not edit the old link.**

**Step 5. A creates the correct link** (Read, 15 minutes, HTTPS only) and B confirms it returns the invoice text.

**Step 6. Compare with what to expect**, then swap roles for the next card.

| Card | Expected status and code | Which field gives it away | Fix |
| --- | --- | --- | --- |
| 1 | 403 `AuthenticationFailed`, and the detail complains about the signed expiry time (wording varies) | `se` is in the past | New link with a later expiry |
| 2 | 403 `AuthorizationPermissionMismatch` | `sp=w` and no `r` | New link with **Read** |
| 3 | 403 `AuthorizationSourceIPMismatch` and the caller's address in the message | `sip=203.0.113.1` is not your address | New link with no IP restriction or with your address |
| 4 | 403, authentication failed (signature does not match) | `sr=b` and the link is for a different blob | A separate link for the other blob, or a container-level SAS if the partner needs many |
| 5 | 403 `AuthenticationFailed`, the detail says the MAC signature is not the same as any computed signature | The edited field | A new link: you cannot repair a signature |

The exact wording of messages can vary slightly. If yours differs, write down both and tell the trainer.

- [ ] Card 1 diagnosed and fixed
- [ ] Card 2 diagnosed and fixed
- [ ] Card 3 diagnosed and fixed
- [ ] Card 4 diagnosed and fixed
- [ ] Card 5 diagnosed and fixed

**Hard mode (optional).** A creates a link with **two** defects (for example a 2 minute expiry **and** write-only). B fixes them one at a time. Notice that the service reports only the first problem it finds, so a fix can reveal the next one.

### Network rules: a valid SAS can still be refused

A SAS grants permission. The storage **firewall** decides whether the caller may reach the account at all. You test this on your own account. **Do this last and put it back.**

1. First generate a fresh good link (Read, 15 minutes, HTTPS only) for `INV-UK-0001.txt` and keep it ready.
2. In the storage account open **Security + networking**, **Networking**.
3. Under **Public network access** choose **Enabled from selected virtual networks and IP addresses**.
4. Do **not** tick **Add your client IP address**. Click **Save**.
5. Wait one minute, then open your good link in an incognito window.

**Expect:** 403 with the code `AuthorizationFailure` and a message that the request is not authorized to perform this operation. The SAS is perfect; the firewall refused the caller before the SAS was looked at.

6. Now tick **Add your client IP address** (or type your public IP in **Address range**) and **Save**. Wait one minute and refresh the link.

**Expect:** the invoice text appears again.

7. **Restore.** Set **Public network access** back to **Enabled from all networks** and **Save**.

- [ ] Firewall on, my link refused with a network error
- [ ] My own IP added, my link works
- [ ] Network access restored to **Enabled from all networks**

With the firewall on you may also be unable to list blobs in the portal until you add your IP or restore access. That is the same mechanism.

**Answer in your notes**

1. Why does editing a SAS link never work?
2. A partner has a valid SAS and still gets `AuthorizationFailure`. What do you check first?
3. Why is a 15 minute, read-only, one-blob link safer than giving the partner the account key?
4. Which error would tell you the link was signed correctly but the caller is in the wrong place?

**Stretch**

1. Create a **container-level** SAS (open the container, **Shared access tokens**) with Read and List for `invoices-uk`. What does `sr` become, and what can the partner now see that they could not before?
2. In the **Generate SAS** tab, compare the **Signing method** options **Account key** and **User delegation key**. Which one does not rely on the account key, and why might NovaTel prefer it? (Day 3 returns to this.)

## Quiz, cleanup and troubleshooting

### Quiz (10 min, closed book)

Answer on paper first, then check below. The reason matters more than the letter.

**Q1.** You want to find every invoice with `status = paid` in a container within seconds. What do you use?

- A. Blob metadata
- B. Blob index tags
- C. Blob properties
- D. The container's access policy

**Q2.** A blob has been in the Cool tier for 10 days and you move it to Archive. What happens?

- A. It reads normally, only slower
- B. It cannot be read until it is rehydrated, which can take hours
- C. It is deleted automatically after 180 days
- D. Nothing, until the next lifecycle run

**Q3.** A lifecycle rule has the prefix `invoices-uk/` and tiers to Cool after 30 days. Today a UK invoice is 40 days old. When is it tiered?

- A. Immediately
- B. At the next daily evaluation of the policy, normally within about a day
- C. Only when someone clicks Run
- D. Never, because the prefix is wrong

**Q4.** A partner edits the `se` value of a SAS link to a later date. What happens?

- A. It works
- B. It fails, because the signature covers every field and no longer matches
- C. It works over HTTP only
- D. It returns 404

**Q5.** After you fix an expired SAS the partner gets `AuthorizationPermissionMismatch`. What is the likely cause?

- A. The firewall blocks them
- B. The link does not include the permission the request needs, such as Read
- C. The container does not exist
- D. TLS is too old

**Q6.** A partner has a perfect SAS and still gets `AuthorizationFailure`. What do you check first?

- A. The blob's tier
- B. The storage account's network rules (firewall)
- C. The file extension
- D. The invoice's metadata

**Answers**

| Q | Answer | Reason |
| --- | --- | --- |
| 1 | B | Index tags are indexed and searchable. Metadata is only returned with a blob you already found. |
| 2 | B | Archive is offline. Rehydrating takes hours and early moves can cost an early-deletion charge. |
| 3 | B | Policies run about once a day and changes can take up to 24 hours to start. |
| 4 | B | The signature is computed over every field. Any edit makes it invalid, so you must issue a new SAS. |
| 5 | B | The signature is valid but the permission field does not allow the operation. |
| 6 | B | `AuthorizationFailure` points to the firewall, a disabled network path or another network rule, not the SAS. |

**Score.** 6 of 6: ready to use lifecycle rules and SAS links at work. 4 or 5: redo the pair task cards you found hard. Below 4: repeat Lab 1 task 5 and Lab 3 card 5, then retake the quiz.

### Clean up at the end of the day

Everything you created sits in your own resource group and carries the tags `course`, `day` and `owner`.

**Option 1: the cleanup script (trainer or anyone with the Day 2 scripts).** In PowerShell, from the folder with the scripts:

```powershell
.\scripts\cleanup-day2.ps1 -Id <id> -ResourceGroup rg-nvtl-oct26-<id> -Owner <id> -Preview
.\scripts\cleanup-day2.ps1 -Id <id> -ResourceGroup rg-nvtl-oct26-<id> -Owner <id>
```

Read the `-Preview` list first. It must show only your storage account.

**Option 2: the portal.**

1. Open your resource group, `rg-nvtl-oct26-<id>`.
2. Tick your storage account `stnovatel<id>`. Check its tags show your `owner`.
3. Click **Delete**, type the account name to confirm, then **Delete**.

**Expect:** the resource group no longer lists the storage account. Do the same for any other resource you created that carries your tags.

**On your computer.** Delete `C:\novatel\invoices` if you wish and any saved SAS links. Close incognito windows.

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Cannot see my resource group | Wrong account, or it has not been created yet | Ask the trainer. Do not make another one. |
| Storage account name rejected or taken | Name must be 3 to 24 lowercase letters and digits and globally unique | Use `stnovatel<id>` or add a digit |
| Upload shows a failure for a file | The blob already exists | Tick **Overwrite if files already exist**, or rename |
| I cannot find where to edit metadata | Portal layout differs | Open the blob, look for **Metadata** on the overview, or use the search box at the top. Tell the trainer. |
| Tag search returns nothing | Tags not saved, or you are in a different container | Save the tags and search in the same container. The portal filter works per container. |
| Archived blob will not download | Archive is offline | This is correct. Rehydrate only if told to, it takes hours. |
| Lifecycle did nothing today | The policy runs about once a day and changes take up to 24 hours | Check tomorrow. Compare **Code view** with the reference JSON. |
| SAS link returns 403 | See the SAS table in Lab 3 | Read `<Code>` and `<Message>`, decode the fields, issue a new link |
| SAS link works for others but not me | The firewall or an IP restriction | Check **Networking** and the link's `sip` field |
| I cannot list blobs in the portal after turning the firewall on | The firewall also blocks your own address | Add your client IP or set **Enabled from all networks** |
| PowerShell script prints an error | Pasted partly, or a curly quote was pasted | Paste the whole script again into a fresh window |

### What has and has not been checked

| Item | Status |
| --- | --- |
| Portal path for lifecycle rules (**Data management**, **Lifecycle management**, **Add a rule**, **Details**, **Base blobs**, **Filter set**, **Code view**), once-a-day evaluation, up to 24 hours for changes, least-expensive action wins | Checked against Microsoft Learn |
| Blob index tags: set on the blob overview, per-container **Blob Index tags filter**; metadata is not searchable and metadata updates change last-modified time | Checked against Microsoft Learn |
| SAS error codes `AuthorizationSourceIPMismatch`, `AuthorizationPermissionMismatch`, `AuthenticationFailed`, `AuthorizationFailure` and their meaning | Checked against Microsoft Learn |
| Exact portal button names and positions (blob **Metadata** edit, **Change tier**, **Generate SAS**, folder field on upload), whether a 1 day lifecycle threshold is accepted, whether an overwrite upload clears metadata and tags | **Not run by me in a live portal.** Check in the dry run. |
| Exact error text for each SAS card, especially cards 4 and 5 | Not run. Expect wording to differ slightly. |
| The PowerShell invoice generator | Not run. Written to be Windows PowerShell 5.1 compatible. |

### Trainer dry run (30 min, the day before)

1. Do Prepare in the trainer resource group with ID `nvtrain`, including the generator.
2. Run Demo 1 and every Lab 1 task. Note any portal label that differs and edit this doc.
3. Create the `demo-tier-aged` rule **two days before class** and confirm the blobs are Cool on the class day.
4. Run all five SAS cards with a colleague and replace the expected error text with what you actually see.
5. Turn the firewall on and off once and confirm your own browser behaves as written.
