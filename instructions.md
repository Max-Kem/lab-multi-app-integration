# Telegram screams, Sheets remembers

> **How you'll submit this lab**
>
> This repo is your lab. Fork it, do the work described below in your fork, then open a pull
> request back into this repository. An AI reviewer will check your PR against `rubric.md` and
> leave feedback directly on the PR. See `README.md` for the full workflow.

## Learning Goal

You will map one source event into one destination record and prove the workflow ran correctly.

## Working Context

A team loses time when requests arrive in one tool and records live in another.
You build a narrow n8n workflow that accepts one source event, transforms only the needed fields, and writes one inspectable destination record.
The workflow should be small enough that another person can trace the event path without opening every node.

## Proof of Learning

- Submit one `lab_proof.md` with the workflow export, one source payload, one destination record, and a short explanation of the mapping.
- The evidence must show the trigger, the transform, and the final record.

## Pseudocode

```text
INPUT: one source event payload
DECIDE: destination fields and required transformations
BUILD: trigger -> normalize fields -> write destination record
VERIFY: inspect the destination record and the execution trace
EXPLAIN: justify the mapping and the operational risk
```

## Prompt Discipline

Use AI for review, debugging, or critique. Do not ask it to produce the final artifact or final answer.

```text
Role: Workflow design reviewer
Task: Review my n8n event-flow plan before I build it.
Inputs: source payload, destination schema, and mapping table.
Constraints: Do not build the workflow for me. Identify brittle fields and missing checks.
Output format: three mapping risks, two verification checks, and one simplification.
Verification: Name the trace or record that proves the workflow ran correctly.
```

## Build

## CFU ladder (checkpoints)

Use this as a quick self-check while you build.

1. **Recognize:** Identify the **trigger** output shape in the execution panel (one sample item’s keys).
2. **Apply:** Create the **destination** container first (base/table/sheet/list) with column names you intend to fill.
3. **Integrate:** Add **Set** assignments so every destination column has a mapped expression or literal.
4. **Verify:** Execute the workflow and confirm the destination row/object matches the trigger content.
5. **Stretch:** Add a guard (IF node, filter, or simple condition) so only relevant events are stored.

---


## Step-by-Step Instructions

### Step 1: Choose your integration pair and scenario

Do: Lock a minimal scope: **one start event** and **one place to write**.

**Everyone**

1. Pick **source**: something that emits JSON into n8n (Telegram Trigger, Webhook, Gmail Trigger, Slack Trigger, etc.).
2. Pick **destination**: something that accepts **create/update** operations (Airtable, Google Sheets, Notion, Postgres, CRM nodes, etc.).
3. Write a **short real-world justification** (3-5 sentences is enough): **who** uses this flow (role or team), **what** triggers it, **where** the data lands, and **why** it beats manual copy-paste or inbox chaos (time saved, fewer dropped requests, audit trail, etc.). Avoid toy labels like “Test bot to random sheet” unless you explain a plausible test plan for a real deployment.
4. In one sentence, state the technical trigger to destination behavior (example: “When a client messages my bot, log the text and sender to a tracker table.”).

**Suggested track (Telegram to Airtable)**
Use this if you want the shortest path: **Telegram Trigger** to **Set** to **Airtable** (create record).

---

### Step 2: Prepare the destination

Do: Create the tabular or structured target **before** mapping.

**Suggested track: Airtable**

1. Open [Airtable](https://airtable.com/) and create a base (example name: **Client requests log**).
2. In the primary table, add columns (example schema):

| Column name   | Type        | Notes                          |
|---------------|-------------|--------------------------------|
| Timestamp     | Date/time   | When n8n received the event    |
| Sender        | Single line | Username or display name       |
| Chat          | Single line | Chat id or title               |
| Message text  | Long text   | Body of the message            |
| Status        | Single line | Default e.g. `New`             |

3. Note the **base**, **table name**, and **field names** exactly: they must match what you send from n8n.

**Other destinations**
Create the sheet, table, database row schema, or Notion database properties first; you’ll reference these names in the Set node and destination node.

---

### Step 3: Sketch the workflow structure

Do: Same skeleton for almost any pair.

**Everyone**

1. Create a new workflow (example name: **Multi-app log: source to destination**).
2. Plan nodes:

```
Source trigger (Telegram / Webhook / …)
  ↓
Set — normalize fields for your destination
  ↓
Destination (Airtable / Sheets / …)
```

3. Add empty nodes in that order and connect them: you’ll configure credentials next.

---

### Step 4: Configure the source (credentials + trigger)

Do: Authenticate and receive **one real execution item** from your source.

**Suggested track: Telegram**

1. In Telegram, open **@BotFather**, create a bot, and copy the **API token**.
2. In n8n, create **Telegram API** credentials with that token.
3. Add a **Telegram Trigger** node:
   - Select your credentials.
   - Limit updates if offered (for testing, **message** updates are enough).
4. Start listening / activate the workflow per your n8n hosting model, send your bot a short test message, and confirm you see output on the trigger node.

**Note:** Telegram allows **one active Telegram Trigger per bot** across workflows: don’t duplicate the same bot token on two workflows.

**Everyone else**
Complete OAuth/API setup for your trigger node and produce **one sample execution** so you can see `$json` keys in the panel.

---

### Step 5: Use a Manual Trigger for dry runs (recommended)

Do: Test mapping **without** depending on the live source while you iterate.

**Everyone**

1. Duplicate the workflow or temporarily add a **Manual Trigger** feeding the same **Set** node (disconnect the live trigger while debugging if needed).
2. Paste **representative JSON** that matches your source. For Telegram-shaped payloads, structure often resembles nested `message` objects; **your** keys depend on the trigger: copy from a real execution when possible.

**Example stub (illustrative only: adjust to your actual `$json`):**

```json
{
  "message": {
    "text": "How do I implement RAG for my documents?",
    "from": { "username": "sample_user", "first_name": "Sample" },
    "chat": { "id": -1001234567890, "title": "Support" }
  }
}
```

3. Run **Execute workflow** from the Manual Trigger and confirm the Set node receives these fields.

**Expected first output:** one item whose fields you can reference with expressions (for example `{{ $json.message.text }}` if your payload nests under `message`).

---

### Step 6: Transform data with the Set node

Do: Produce **flat** keys that match your destination field names.

**Everyone**

1. Open the **Set** node (manual assignment mode is fine).
2. For each destination column/property, add an assignment:

**Suggested track: example mappings for Telegram to Airtable**

| Destination field | Example expression |
|-------------------|--------------------|
| Timestamp | `{{ $now }}` |
| Sender | `{{ $json.message.from.username }}` (add a fallback expression if username is often empty) |
| Chat | `{{ String($json.message.chat.id) }}` or chat title if present |
| Message text | `{{ $json.message.text }}` |
| Status | `New` |

3. Run once and inspect output: field names should mirror Airtable **exactly** (case-sensitive).

If your real Telegram payload differs, **do not guess**: pin open the trigger output and copy paths from the execution data.

---

### Step 7: Configure the destination node

Do: Create or append records using the Set node output.

**Suggested track: Airtable**

1. Add an **Airtable** node after **Set**.
2. Choose operation **Create** (or **Append** depending on n8n version/UI).
3. Select base/table and map fields using the flattened keys from Set (for example `{{ $json['Message text'] }}` if your Set node used that label).

**Everyone**

- Match **operation** to what you built (append row, insert row, create page, insert record).
- Re-run authentication if you see permission errors; confirm scopes/tokens include write access.

---

### Step 8: Test end-to-end

Do: Prove data lands where users expect it.

**Everyone**

1. Execute from **Manual Trigger** first: confirm the destination updates.
2. Switch back to your **live trigger**, send a real event (message, form submit, etc.).
3. Open the destination UI and verify a new row matches the event.

**Expected result:** at least one record showing message/content, sender/source identifier, and timestamp consistent with your mapping.

---

### Step 9 (Optional): Hardening

Do: Move from demo to tolerable prototype quality.

**Ideas:**

- Add an **IF** node to skip empty bodies or bot-only noise.
- Store secrets only in n8n credentials: never hard-code tokens in expressions.
- Rename the workflow and add a short note in the description (source, destination, owner).

---

## Submission Guidelines


## Extension Activities (Optional)

1. **Filter:** Only log messages containing keywords or commands.
2. **Enrich:** Add a second destination (e.g. Slack notification) in parallel: mind rate limits.
3. **Deduplicate:** Use a simple cache or lookup before inserting (advanced).
4. **AI assist:** Summarize long messages before storing (uses extra API cost: optional).

---

## Resources

- [n8n integrations library](https://docs.n8n.io/integrations/)
- [Expressions](https://docs.n8n.io/code/expressions/)
- Suggested path: [Telegram Trigger](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger/), [Airtable](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/)

---

## Troubleshooting

**Issue:** Destination shows empty fields
**Solution:** Compare Set output keys to destination field names character-for-character; watch for nested paths (`message.text` vs `text`).

**Issue:** Authentication fails on one side
**Solution:** Regenerate tokens in the provider UI; ensure OAuth redirect URLs match n8n’s docs for cloud vs self-hosted.

**Issue:** Telegram Trigger never fires
**Solution:** Confirm the workflow is active, the bot token matches, and you aren’t using the same bot on two active Telegram Trigger workflows.

**Issue:** Airtable returns permission or table errors
**Solution:** Verify base/table names, PAT scopes, and that field types accept the values you send (long text vs single line).

**Issue:** Different integration pair than the suggested path
**Solution:** Read your trigger node’s execution payload first; map **after** you see real keys: generic tutorials won’t match every node.

## Verify

The reviewer can inspect one execution trace and one destination record that contains the expected mapped fields.

## Explain

Which field is most likely to break when the source payload changes, and how would you detect it?

## Submission Sanity Check

Before you submit:
- Keep only files for this lab.
- Name files clearly.
- Remove secrets, API keys, tokens, and unrelated data.
- Make `lab_proof.md` the entry point for the reviewer.

## Optional Stretch

- Add one operational control: routing, duplicate detection, error notification, retry handling, or a second destination field for source id.
- Create a small UI with vibecoding only if it safely submits or previews the event payload.
- Deploy the UI only if it can run without exposing credentials.
