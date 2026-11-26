# Mini Spec: Integration Events Demo (Microfinance Scenario)

## Goal

- Demonstrate key integration project tasks in one compact build:
  - Azure Service Bus pub/sub
  - Standardized message contract
  - Dead-lettering and retries
  - Azure Functions and Logic App flows
  - Application Insights end-to-end telemetry
- Runs on Azure using simulated D365 and ServiceNow sources; the core integration architecture is production-shaped.

---

## 1. Scenario

- A microfinance firm tracks customers in a simulated **D365** instance and support incidents in a simulated **ServiceNow** instance.
- Both systems publish state changes as events to a single Azure Service Bus topic.

---

## 2. Architecture & Components

- **Fake D365 & Fake ServiceNow:** Two lightweight event producers (REST client `.http` file or small timer trigger) calling the intake endpoint with `customer-updated` and `incident-created` events.
- **Intake Function (HTTP Trigger):** Webhook ingestion endpoint.
  - Validates the API/function key.
  - Validates the incoming payload schema.
  - Wraps the payload in the standard message envelope and publishes it to the Service Bus topic.
  - Returns `202 Accepted` for valid requests and `400 Bad Request` for invalid input.
- **Service Bus Topic (`integration-events`):** Provisioned inside the existing `DevPulse` Service Bus namespace.
- **Resource Organization:** Place all newly created Azure Functions, Logic Apps, and supporting resources in a dedicated Resource Group.

---

## 3. Message Envelope

All events share the exact same top-level schema:

```json
{
  "messageId": "guid",
  "eventType": "customer-updated | incident-created | incident-escalated",
  "version": "1.0",
  "source": "d365 | servicenow | durable-orchestrator",
  "occurredAtUtc": "2026-11-27T00:00:00Z",
  "payload": {}
}
```

- **Payload Design:** Keep payloads minimal (customer/incident ID plus only the fields required by downstream consumers).
- **Schema Evolution:** Additive changes only—add new fields, never rename or remove existing ones.
- **Routing Metadata:** Set `eventType` as a Service Bus message application property so topic subscriptions can filter natively.

---

## 4. Subscriptions

Each subscription maintains its own Dead-Letter Queue (DLQ):

- **`customers-to-cosmos`**
  - **Filter:** `eventType = 'customer-updated'`
  - **Consumer:** Azure Function that upserts customer records into Cosmos DB, keyed by `messageId`.
- **`incidents-to-logicapp`**
  - **Filter:** `eventType = 'incident-created'`
  - **Consumer:** Azure Logic App with a Service Bus trigger that calls the customer API or dispatches a notification.
- **`incidents-durable` _(Optional / Phase 2)_**
  - **Filter:** `eventType = 'incident-created'`
  - **Consumer:** Durable Functions starter (see Section 7).

---

## 5. Reliability & Resilience

- **Explicit Message Settlement:** Disable auto-complete (`autoCompleteMessages: false`). Complete messages only after downstream work succeeds.
- **Retry & DLQ Policy:** Set `MaxDeliveryCount = 5`. Failed messages automatically move to the DLQ with an explicit dead-letter reason.
- **Idempotency:** Consumers upsert using `messageId` and verify `occurredAtUtc` so out-of-order/older events never overwrite newer state.
- **Dead-Letter Demo Flow:**
  1. Inject a malformed message.
  2. Inspect the dead-letter reason in the DLQ.
  3. Fix the payload and replay a corrected copy back to the topic.
- **Outbound HTTP Resilience:** Use the standard .NET resilience pipeline (`Microsoft.Extensions.Http.Resilience`) for retries, timeouts, and circuit breakers.

---

## 6. Telemetry & Monitoring

- **Distributed Tracing:** Connect Application Insights to the Intake Function, consumer Functions, and Logic App.
- **Correlation:** Propagate `messageId` and `operation_Id` via Service Bus message properties so a single KQL query traces an event from HTTP intake to Cosmos DB persistence.
- **Dashboard:** Single Azure Dashboard displaying failures, dead-letter message counts, and end-to-end processing latency.
- **Alerting:** Azure Monitor alert triggered whenever dead-letter message count `> 0`.

---

## 7. Durable Functions _(Optional — Build Last)_

- **Starter Function:** Listens to `incidents-durable` and starts an orchestration using `incidentId` as the instance ID (preventing duplicate workflows).
- **Orchestrator Workflow:**
  1. Call activity to assign an owner and log the assignment.
  2. Wait for an external approval event **or** a 30-second durable timer.
  3. On timeout, escalate by writing to Cosmos DB and publishing an `incident-escalated` event to the topic.

---

## 8. Tech Stack

- **Compute:** .NET 8 Isolated Worker Azure Functions (or .NET 10 for single-version alignment across projects).
- **Infrastructure as Code:** Bicep for provisioning new Azure resources.
- **CI/CD _(Optional)_:** GitHub Actions using OIDC authentication.

---

## 9. Build Order (Incremental Delivery)

- **Day 1:** Intake Function, `integration-events` topic, `customers-to-cosmos` consumer, and one simulated source.
- **Day 2:** Second simulated source, `incidents-to-logicapp` subscription, and Logic App workflow.
- **Day 3:** Dead-letter demo & replay script, App Insights KQL correlation, dashboard, and DLQ alert.
- **Phase 2:** Durable Functions escalation flow, `README.md` with architecture diagram, and a 2-minute walkthrough demo.

---

## 10. Out of Scope

- Live integrations with real D365 or ServiceNow environments
- Frontend UI
- Authentication beyond Function/API keys
- Load or performance testing
