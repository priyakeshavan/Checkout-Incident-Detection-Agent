# Checkout Incident Detection Agent

**An AI-assisted checkout event triage and alerting prototype built with Zapier, Google Sheets, AI by Zapier, and Webhooks by Zapier.**

## Overview

This project demonstrates a no-code workflow that reads new checkout events, checks a processing checkpoint, retrieves a checkout-step baseline, asks an AI step to classify incident severity, checks for an existing command, writes an alert or log command, posts a webhook, and advances the checkpoint.

A **live end-to-end test with EVT010** created a `HIGH` / `ALERT` command, sent a webhook request, and updated the processing timestamp. The Zap was later disabled when the Pro trial expired; the repository documents the implementation and historical test evidence, not a currently running service.

## Problem

E-commerce checkout issues need consistent triage. A simple rules-guided AI workflow can help flag important events and record why an alert was issued. This project is a learning prototype rather than a production incident monitoring system.

## Tools

- **Zapier:** orchestration, filters, Google Sheets actions, and webhook POST
- **AI by Zapier:** rule-guided classification and explanation
- **Google Sheets:** event input, baseline metrics, command log, and processing checkpoint
- **Webhook.site:** temporary endpoint for testing webhook delivery

## Workflow architecture

![Complete 11-step Zapier workflow](screenshots/01-workflow-architecture.png)

1. **New Spreadsheet Row** — read a new event from `cart_events`.
2. **Lookup Spreadsheet Row** — read `last_processed_timestamp` from `state`.
3. **Filter** — continue only if the event timestamp is newer than the checkpoint.
4. **Lookup Spreadsheet Row** — retrieve the `baseline_rate` for the event's checkout step.
5. **AI by Zapier** — classify the event and explain the action.
6. **Lookup Spreadsheet Row** — search `agent_commands` for the event ID.
7. **Filter** — proceed only when an existing command was not found.
8. **Create Spreadsheet Row** — write the decision to `agent_commands`.
9. **Webhooks POST** — send the command payload to a test endpoint.
10. **Lookup Spreadsheet Row** — locate the state row.
11. **Update Spreadsheet Row** — store the latest processed event timestamp.

## Data structure

| Sheet | Purpose | Main columns |
| --- | --- | --- |
| `cart_events` | Checkout input | `event_id`, `timestamp`, `checkout_step`, `cart_value`, `device` |
| `baseline_metrics` | Reference rates | `checkout_step`, `baseline_rate` |
| `agent_commands` | Decision log | `command_id`, `action`, `severity`, `reason`, `checkout_step`, `cart_value`, `device`, `baseline_rate`, `timestamp` |
| `state` | Processing checkpoint | `key`, `value` |

![Input events](screenshots/02-input-events.png)
![Baseline metrics](screenshots/03-baseline-metrics.png)

## Decision rules

The AI step was prompted to produce **HIGH / ALERT** when:
- The checkout step is **Payment**, or
- The cart value is **1,000 or more**.

The prompt also describes rules for an elevated current failure rate versus baseline and for three or more mobile failures within ten minutes. **Those two aggregate rules were not fully implemented**, because the workflow did not supply a calculated current failure rate or reliable recent-event history. Other events are intended to produce **LOW / LOG**, but the saved evidence demonstrates ALERT cases, not a verified LOG live run.

![AI decision preview for EVT008](screenshots/04-ai-decision.png)

## Duplicate check

The Zap looks up `command_id` in `agent_commands` using the incoming event ID. A filter permits processing only if the lookup does not find a match. This is a practical best-effort check, **not an atomic exactly-once guarantee** for concurrent executions.

![Duplicate lookup](screenshots/05-duplicate-lookup.png)
![Duplicate filter](screenshots/05b-duplicate-filter.png)

## Verified results

| Event | Context | Observed result |
| --- | --- | --- |
| EVT008 | Shipping, cart value 1,200 | `HIGH` / `ALERT` in test output and command sheet |
| EVT010 | Payment, baseline rate 0.02 | Live run created `HIGH` / `ALERT` command, sent webhook request, and advanced checkpoint to `2026-10-08 09:40:00` |

![Recorded commands](screenshots/06-agent-commands.png)

The following screenshot shows the **outbound payload from Zapier's Step 9 run details**, with the webhook endpoint redacted. It documents the request data; delivery was also observed during the live test.

![Webhook POST request data](screenshots/07-webhook-request.png)

![State after EVT010](screenshots/08-state-update.png)

## Known limitations and improvements

- **Incomplete field mapping:** `cart_value` and `device` were blank for EVT010 in `agent_commands` and the outbound webhook payload, even though they existed in `cart_events`.
- **Baseline inconsistency:** The AI preview for EVT008 returned a baseline failure rate of `0`, while the Shipping baseline lookup and recorded command showed `0.03`. The AI mapping needs correction.
- **Aggregate rules not operational:** Failure-rate comparisons and repeated mobile failures require historical events and reliable aggregation.
- **Checkpoint trade-off:** Timestamp filtering may skip out-of-order or older unprocessed events.
- **Concurrency/retries:** A lookup plus filter does not guarantee exactly-once processing or atomic coordination between the sheet write, webhook, and state update.
- **Trial dependency:** The Zap uses paid Zapier features and is currently disabled after the Pro trial. The workflow is preserved as a documented prototype.
- **Testing scope:** One live event (EVT010) was verified end to end; the full set of original input events was not automatically replayed or exhaustively validated.

## Repository contents

```text
checkout-incident-detection-agent/
├── README.md
└── screenshots/
    ├── 01-workflow-architecture.png
    ├── 02-input-events.png
    ├── 03-baseline-metrics.png
    ├── 04-ai-decision.png
    ├── 05-duplicate-lookup.png
    ├── 05b-duplicate-filter.png
    ├── 06-agent-commands.png
    ├── 07-webhook-request.png
    └── 08-state-update.png
```

## Privacy and reproducibility

The screenshots contain demonstration data, not production customer information. Webhook endpoints and any credentials should be redacted before public sharing. This repository documents a no-code Zapier configuration rather than exporting an executable Zap or exposing secrets. Rebuilding it requires recreating the Google Sheets tabs and Zapier steps with your own accounts and test webhook endpoint.
