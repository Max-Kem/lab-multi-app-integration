# LAB Proof — Slack to Airtable Reservation Intake

**Lab:** Telegram screams, Sheets remembers — adapted integration pair: **Slack → Airtable**  
**Workflow:** `Restaurant Reservation Intake`  
**Testing date:** 2026-11-03  
**Status:** End-to-end test successful; workflow subsequently unpublished (`active: false` in export).

## 1. Scenario and business justification

A small restaurant receives table reservation inquiries through a dedicated Slack channel. Front-of-house staff need a structured overview of incoming requests instead of searching through chat history and copying information manually. The workflow automatically captures each message in a fixed reservation format, extracts the requested date, time, and party size, and stores a new record in Airtable for follow-up. The saved record provides an inspectable intake log and reduces the chance that a request is overlooked; it does **not** confirm table availability or a booking.

**Technical behavior (one sentence):** When a user posts a formatted reservation request in `#restaurant-reservations`, n8n receives the Slack event, normalizes seven destination fields, and creates one `Pending` record in Airtable.

## 2. Workflow structure and configuration

```text
Slack Trigger (New Message Posted to Channel)
        ↓
Edit Fields (Set / Manual Mapping)
        ↓
Airtable (Record → Create; Typecast enabled)
```

- **Source:** Slack app `Restaurant Reservation Bot`, isolated test workspace, `#restaurant-reservations` channel, monitored by channel ID.
- **Destination:** Airtable base `Restaurant Reservations`, table `Reservation Requests`.
- **Workflow nodes:** Slack Trigger (`n8n-nodes-base.slackTrigger`), Edit Fields (`n8n-nodes-base.set`), Airtable (`n8n-nodes-base.airtable`).
- **Destination schema:** `Customer` (text), `Reservation Date` (date), `Reservation Time` (text), `Guests` (number), `Message` (long text), `Received At` (date with time), `Status` (text).
- **Credentials:** Stored in n8n credential storage, not included in this document.
- **Current deployment state:** Unpublished following a short test. Production-grade Slack signature verification was **not** functioning in the tested setup; see operational risks.

## 3. Source event — real Slack payload

The following event was received from Slack during the first test on **2026-11-03**. The event shape and business values are preserved; personal/workspace identifiers have been redacted.

```json
{
  "type": "message",
  "user": "REDACTED_USER",
  "ts": "1793723723.027899",
  "client_msg_id": "REDACTED_CLIENT_MSG_ID",
  "text": "RESERVATION | 2026-11-07 | 19:00 | 4",
  "team": "REDACTED_TEAM",
  "blocks": [
    {
      "type": "rich_text",
      "elements": [
        {
          "type": "rich_text_section",
          "elements": [
            {
              "type": "text",
              "text": "RESERVATION | 2026-11-07 | 19:00 | 4"
            }
          ]
        }
      ]
    }
  ],
  "channel": "REDACTED_CHANNEL",
  "event_ts": "1793723723.027899",
  "channel_type": "channel"
}
```

**Input interpretation:** The message is stored in the top-level `text` field, the sender in `user`, and Slack's event timestamp in `ts`. The fields are **not** nested under an `event` object in this n8n output. The `blocks` array also contains Slack rich-text formatting, but mapping uses `text` to avoid unnecessary nesting.

## 4. Field mapping and transformations

The Edit Fields node is in **Manual Mapping** mode, with **Include Other Input Fields disabled**, so its output contains only the seven destination fields.

| Slack input / source | Airtable destination | n8n transformation | Reason |
|---|---|---|---|
| `user` | `Customer` | `{{ $json.user }}` | Retain a traceable source identifier without guessing a display name. |
| `text` | `Reservation Date` | `{{ $json.text.split("|")[1].trim() }}` | Take the second pipe-separated section and remove whitespace. |
| `text` | `Reservation Time` | `{{ $json.text.split("|")[2].trim() }}` | Take the third section as the requested time. |
| `text` | `Guests` | `{{ Number($json.text.split("|")[3].trim()) }}` | Take the fourth section and convert it to a number. |
| `text` | `Message` | `{{ $json.text }}` | Preserve the original request for auditing. |
| `ts` | `Received At` | `{{ DateTime.fromSeconds(Number($json.ts)).toISO() }}` | Convert Slack Unix seconds to an ISO 8601 timestamp. |
| Literal | `Status` | `Pending` | New requests are not automatically confirmed reservations. |

**Required incoming format:** `RESERVATION | YYYY-MM-DD | HH:mm | guests`.

For the first sample event, the normalized values were:

```json
{
  "Customer": "REDACTED_USER",
  "Reservation Date": "2026-11-07",
  "Reservation Time": "19:00",
  "Guests": 4,
  "Message": "RESERVATION | 2026-11-07 | 19:00 | 4",
  "Received At": "2026-11-03T16:35:23.027Z",
  "Status": "Pending"
}
```

*Note: `Customer` is redacted consistently in the source/output examples. The actual record contained the Slack user ID.*

## 5. Destination record — Airtable verification

Airtable's `Create a record` node successfully returned a created record for the **four-guest** sample, and the record was visible in Airtable. Below is the observed set of destination field values, with the customer identifier redacted:

```json
{
  "fields": {
    "Customer": "REDACTED_USER",
    "Reservation Date": "2026-11-07",
    "Reservation Time": "19:00",
    "Guests": 4,
    "Message": "RESERVATION | 2026-11-07 | 19:00 | 4",
    "Received At": "2026-11-03T16:35:23.027Z",
    "Status": "Pending"
  }
}
```

- **Created in:** `Restaurant Reservations` → `Reservation Requests`.
- **Airtable result:** Successful node execution with a returned record ID and `createdTime` (identifiers omitted here).
- **Type conversion:** Initial Airtable execution rejected `Reservation Date`; enabling the Airtable node's **Typecast** option resolved the error. The target field remained an Airtable date field with time disabled.
- **Timestamp:** `Received At` is shown in UTC (`Z`) in the API output and in local time in the Airtable user interface. These represent the same instant.

## 6. Full live execution trace — source to destination

A **second Slack message** was sent *after* the mapping was completed and the workflow was temporarily published:

```text
RESERVATION | 2026-11-08 | 18:30 | 2
```

The following results were verified directly in n8n and Airtable:

| Evidence | Observed result |
|---|---|
| n8n execution ID | `#1453` |
| Execution date/time | 2026-11-03, 19:08:12 (displayed local time) |
| Execution status | **Succeeded** |
| Execution duration | **672 ms** |
| Slack Trigger | Executed successfully; event received |
| Edit Fields | Executed successfully; one transformed item |
| Airtable — Create a record | Executed successfully; one record created |
| Airtable record values | `Reservation Date = 2026-11-08`, `Reservation Time = 18:30`, `Guests = 2`, `Status = Pending` |
| Airtable UI | New reservation row visible; received time shown as approximately 19:08 local time |

**Evidence distinction:** Section 3 contains the actual JSON from the first Slack message (four guests). The *separate* live end-to-end execution `#1453` used the second message (two guests). The full raw JSON for the second event was not separately copied into this report; its input text, successful execution, and destination values were observed in the n8n/Airtable UIs.

**Result:** The source event reached the destination automatically; the successful execution trace and matching Airtable row prove the trigger → transform → create-record path.

## 7. Operational risk and change detection

**Most fragile field:** `text`. The mapping relies on a fixed pipe-separated message structure. If a message lacks a delimiter, changes field order, or contains a non-numeric guest count, `split(...)[n]` can return an unexpected value or `Guests` can become invalid. A change would be detected by reviewing the Edit Fields output, n8n execution errors, and the resulting Airtable row; a production implementation should validate the prefix, section count, date/time, and numeric guest count with an IF/validation guard before insertion.

**Other operational concerns:**

- **Duplicates:** Replayed or repeated Slack messages can create multiple Airtable records. A production system should save a Slack event/message identifier and check it before creating a record.
- **Access and privacy:** The Airtable record contains a Slack user identifier and message text; use minimal scopes, controlled access, and an appropriate retention policy.
- **Authentication:** The Slack signing secret could not be enabled successfully during this lab's webhook verification. The test therefore used an unsigned webhook on the shared n8n environment. This configuration should **not** be deployed for real customer data until the issue is fixed and signature verification is enabled.
- **Booking semantics:** `Pending` means only that a request was logged. The workflow does **not** check table capacity, prevent conflicting reservations, or send an availability confirmation.

## 8. Reviewer verification and conclusion

A reviewer can trace **Slack message → source JSON (`text`, `user`, `ts`) → seven Edit Fields assignments → Airtable Create → visible destination record**, and can verify the complete run in n8n execution `#1453`. The workflow was unpublished after testing. Secrets are not included in this file.

**Lab objective achieved:** One real source event was mapped into one inspectable destination record; a successful end-to-end execution was observed and documented.

## Appendix A — Sanitized n8n workflow export

The following JSON is derived from the actual n8n export. API tokens/credentials, Slack user/workspace/channel IDs, Airtable base/table IDs, internal node/workflow identifiers, and pinned user data were removed or replaced with obvious placeholders. **It documents the real workflow configuration but must be relinked to the relevant resources and credentials before importing or running.**

```json
{
  "name": "Restaurant Reservation Intake",
  "nodes": [
    {
      "parameters": {
        "trigger": [
          "message"
        ],
        "channelId": {
          "__rl": true,
          "value": "REDACTED_CHANNEL_ID",
          "mode": "id"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.slackTrigger",
      "typeVersion": 1,
      "position": [
        -272,
        0
      ],
      "name": "Slack Trigger"
    },
    {
      "parameters": {
        "operation": "create",
        "base": {
          "__rl": true,
          "value": "REDACTED_BASE_ID",
          "mode": "list",
          "cachedResultName": "Restaurant Reservations"
        },
        "table": {
          "__rl": true,
          "value": "REDACTED_TABLE_ID",
          "mode": "list",
          "cachedResultName": "Reservation Requests"
        },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Guests": "={{ $json.Guests }}",
            "Customer": "={{ $json.Customer }}",
            "Reservation Date": "={{ $json['Reservation Date'] }}",
            "Reservation Time": "={{ $json['Reservation Time'] }}",
            "Message": "={{ $json.Message }}",
            "Received At": "={{ $json['Received At'] }}",
            "Status": "={{ $json.Status }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "Customer",
              "displayName": "Customer",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true,
              "display": true,
              "type": "string",
              "readOnly": false,
              "removed": false
            },
            {
              "id": "Reservation Date",
              "displayName": "Reservation Date",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true,
              "display": true,
              "type": "dateTime",
              "readOnly": false,
              "removed": false
            },
            {
              "id": "Reservation Time",
              "displayName": "Reservation Time",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true,
              "display": true,
              "type": "string",
              "readOnly": false,
              "removed": false
            },
            {
              "id": "Guests",
              "displayName": "Guests",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true,
              "display": true,
              "type": "number",
              "readOnly": false,
              "removed": false
            },
            {
              "id": "Message",
              "displayName": "Message",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true,
              "display": true,
              "type": "string",
              "readOnly": false,
              "removed": false
            },
            {
              "id": "Received At",
              "displayName": "Received At",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true,
              "display": true,
              "type": "dateTime",
              "readOnly": false,
              "removed": false
            },
            {
              "id": "Status",
              "displayName": "Status",
              "required": false,
              "defaultMatch": false,
              "canBeUsedToMatch": true,
              "display": true,
              "type": "string",
              "readOnly": false,
              "removed": false
            }
          ],
          "attemptToConvertTypes": false,
          "convertFieldsToString": false
        },
        "options": {
          "typecast": true
        }
      },
      "type": "n8n-nodes-base.airtable",
      "typeVersion": 2.2,
      "position": [
        352,
        0
      ],
      "name": "Create a record"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "e37d4183-6832-4ea4-821f-4edae65d669e",
              "name": "Customer",
              "value": "={{ $json.user }}",
              "type": "string"
            },
            {
              "id": "9def6d72-9baa-472d-8f9a-64af2d49197b",
              "name": "Reservation Date",
              "value": "={{ $json.text.split(\"|\")[1].trim() }}",
              "type": "string"
            },
            {
              "id": "cede3455-5ef8-4475-9ae2-75c9d5bd76eb",
              "name": "Reservation Time",
              "value": "={{ $json.text.split(\"|\")[2].trim() }}",
              "type": "string"
            },
            {
              "id": "f4586976-d30a-4ec8-8908-5b7524dcfc08",
              "name": "Guests",
              "value": "={{ Number($json.text.split(\"|\")[3].trim()) }}",
              "type": "number"
            },
            {
              "id": "f18c2dc6-d580-46e1-96e4-aea62df0a3d5",
              "name": "Message",
              "value": "={{ $json.text }}",
              "type": "string"
            },
            {
              "id": "96342ba8-fd25-455f-acd6-f80ef5915f3c",
              "name": "Received At",
              "value": "={{ DateTime.fromSeconds(Number($json.ts)).toISO() }}",
              "type": "string"
            },
            {
              "id": "51c3f809-7d7f-49b9-8980-48af2f6ed338",
              "name": "Status",
              "value": "Pending",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [
        48,
        0
      ],
      "name": "Edit Fields"
    }
  ],
  "pinData": {},
  "connections": {
    "Slack Trigger": {
      "main": [
        [
          {
            "node": "Edit Fields",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Edit Fields": {
      "main": [
        [
          {
            "node": "Create a record",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1",
    "binaryMode": "separate"
  },
  "tags": []
}
```
