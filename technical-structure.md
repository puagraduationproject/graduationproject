# Technical Structure

> Companion to the project spec. This file covers the code layout, data models, events and tool contracts for every module.
> Status: design document. Living file.

---

## 0. Overview

A **modular monolith**: one backend, one database, strict module boundaries. Not microservices.

```
+-------------------------------------------------------+
|  AI layer                                             |
|  Customer agent  |  Owner copilot  |  Business brain  |
+----------------------+--------------------------------+
                       | tools only, never direct DB
+----------+-----------+----------+-----------+---------+
|   ERP    |    CRM    | Channels | Marketing |  Brand  |
| offerings| contacts  | WhatsApp | campaigns |   kit   |
| stock    | identities| Instagram| content   | store   |
| orders   | timeline  | store msg| follow-ups| page gen|
| payments | stats     | webhooks | ads, GEO  |         |
| shipping | segments  | voice    |           |         |
| expenses | pipelines | templates|           |         |
+----------+-----------+----------+-----------+---------+
+-------------------------------------------------------+
|  Reports: event consumers into summary tables         |
+-------------------------------------------------------+
|  Core: businesses, users, roles, permissions, geo,    |
|  template packs, field registry, layouts, audit,      |
|  outbox, one PostgreSQL database                      |
+-------------------------------------------------------+
```

### The five rules

1. **One owner per piece of data.** Others reference it by ID, never copy it
2. **Modules talk through events.** No module touches another's tables
3. **Reading across modules is a service call**
4. **The AI acts only through tools.** No direct database access
5. **Every write is audited, every state change emits an event**

---

## 1. Repository layout

```
repo/
├── apps/
│   ├── api/              NestJS backend
│   ├── web/              Next.js dashboard + generated store pages
│   └── ai/               Python FastAPI AI service
├── packages/
│   ├── schemas/          shared Zod schemas and types
│   ├── ui/               component library
│   └── config/           eslint, tsconfig, tailwind preset
├── templates/            business type template packs (versioned JSON)
│   ├── clothing-brand.json
│   ├── home-food.json
│   └── service-business.json
└── infra/                migrations, docker, CI
```

### Backend module layout

```
apps/api/src/
├── core/
├── modules/
│   ├── erp/
│   ├── crm/
│   ├── channels/
│   ├── marketing/
│   ├── brand/
│   ├── founder/
│   └── reports/
├── shared/
│   ├── events/           outbox, publisher, listener base
│   ├── tools/            AI tool registry
│   └── audit/
└── main.ts
```

---

## 2. Core

Foundation. Everything sits on this.

### 2.1 Code structure

```
apps/api/src/core/
├── businesses/
│   ├── business.entity.ts
│   ├── business-settings.entity.ts
│   └── business.service.ts
├── users/
│   ├── user.entity.ts
│   ├── membership.entity.ts        (user belongs to business with a role)
│   ├── role.entity.ts
│   ├── permission.service.ts
│   └── auth/
├── geo/
│   ├── geo-area.entity.ts          (governorate, city, district)
│   └── geo.service.ts
├── templates/
│   ├── template-pack.ts            (loader for templates/*.json)
│   ├── capability.service.ts       (resolves businessCapabilities)
│   └── template.service.ts         (applies a pack on signup)
├── fields/
│   ├── field-definition.entity.ts  (the field registry)
│   └── field.service.ts
├── layouts/
│   ├── layout.entity.ts            (page layout JSON per object per template)
│   └── layout.service.ts
├── audit/
│   ├── audit-log.entity.ts
│   └── audit.service.ts
├── events/
│   ├── outbox.entity.ts
│   ├── event-publisher.ts
│   └── event-dispatcher.ts         (reads outbox, dispatches to listeners)
└── notifications/
    ├── notification.entity.ts
    └── notification.service.ts
```

### 2.2 Data model

```
business          id, name, slug, business_type, currency, timezone,
                  capabilities(jsonb), settings(jsonb), created_at

user              id, email, phone, name, password_hash, locale

membership        id, business_id, user_id, role, invited_by, joined_at
                  UNIQUE (business_id, user_id)

role              business_id, name, permissions(jsonb)
                  defaults: owner, manager, sales, support, viewer

geo_area          id, parent_id, type(governorate|city|district),
                  name_ar, name_en, country_code

field_definition  id, business_id, object_type, key, label_ar, label_en,
                  data_type, options(jsonb), required, visible_to_roles[],
                  from_template

layout            id, business_id, object_type, template_key,
                  layout(jsonb), version

audit_log         id, business_id, actor_type(user|ai|system), actor_id,
                  action, object_type, object_id,
                  before(jsonb), after(jsonb), created_at

outbox            id, business_id, event_type, payload(jsonb),
                  created_at, dispatched_at, attempts, last_error

notification      id, business_id, user_id, type, title, body,
                  ref_type, ref_id, read_at, created_at
```

Every business-scoped table has `business_id` with a row-level security policy.

### 2.3 Capability resolution

Resolved once per request into a `businessCapabilities` object that both the UI and the AI tool registry read.

```
capabilities = merge(
  template_pack.capabilities,     // defaults for the business type
  business.capabilities           // overrides the business made
)
```

If `capabilities.inventory` is false:
- the Products page hides stock columns
- the AI tool registry does not expose `check_stock`
- stock movement listeners are not registered

### 2.4 Template pack format

```json
{
  "key": "clothing_brand",
  "version": 1,
  "name_ar": "براند ملابس",
  "name_en": "Clothing brand",
  "capabilities": {
    "inventory": true, "variants": true, "shipping": true,
    "cod": true, "bookings": false, "deals": false, "subscriptions": false
  },
  "offering_types": ["product"],
  "variant_attributes": ["size", "color"],
  "pipelines": {
    "customer": ["new", "ordered", "repeat", "inactive"]
  },
  "fields": [
    { "object": "variant", "key": "fabric", "type": "text", "label_ar": "الخامة" }
  ],
  "layouts": { "order": "...", "contact": "..." },
  "kpis": ["revenue", "orders_count", "top_products", "profit"],
  "ai": {
    "tone": "friendly_egyptian",
    "allowed_tools": ["check_stock", "get_price", "create_order", "get_shipping_fee"],
    "requires_approval": ["send_campaign", "apply_discount"]
  },
  "checklist": ["brand_ready", "store_live", "first_sale", "first_10_customers"]
}
```

### 2.5 Events infrastructure

The **outbox pattern**: a write and its event are committed in the same transaction, so events are never lost.

```
1. service writes the row AND inserts into outbox, one transaction
2. event-dispatcher polls undispatched outbox rows
3. dispatches to registered listeners via BullMQ
4. marks dispatched_at, retries on failure
```

---

## 3. ERP

### 3.1 Code structure

```
apps/api/src/modules/erp/
├── catalog/
│   ├── offering.entity.ts
│   ├── variant.entity.ts
│   ├── category.entity.ts
│   ├── catalog.service.ts
│   └── catalog.controller.ts
├── inventory/
│   ├── location.entity.ts
│   ├── stock-movement.entity.ts
│   ├── stock-level.entity.ts        (cached sum)
│   ├── inventory.service.ts         (applyMovement, reserve, release, rebuild)
│   └── inventory.controller.ts
├── orders/
│   ├── order.entity.ts
│   ├── order-line.entity.ts
│   ├── order-event.entity.ts
│   ├── order-state-machine.ts       (single source of transitions)
│   ├── order-number.service.ts      (per-business numbering)
│   ├── order.service.ts
│   └── order.controller.ts
├── payments/
│   ├── payment.entity.ts
│   ├── refund.entity.ts
│   ├── invoice.entity.ts
│   ├── payments.service.ts
│   └── providers/
│       ├── payment-provider.interface.ts
│       ├── cod.provider.ts
│       └── paymob.provider.ts
├── shipping/
│   ├── shipment.entity.ts
│   ├── tracking-event.entity.ts
│   ├── zone.entity.ts
│   ├── cod-settlement.entity.ts
│   ├── shipping.service.ts
│   ├── cod-reconciliation.service.ts
│   └── carriers/
│       ├── carrier.interface.ts
│       └── bosta.carrier.ts
├── finance/
│   ├── expense.entity.ts
│   ├── ledger-entry.entity.ts
│   └── finance.service.ts
├── erp.events.ts
├── erp.tools.ts
└── erp.module.ts
```

### 3.2 Data model

**Catalog**
```
offering       id, business_id, type(product|service|subscription),
               name, description, category_id, custom_fields(jsonb), active

variant        id, offering_id, name, sku, attributes(jsonb),
               price_cents, cost_cents, currency, active

category       id, business_id, name, parent_id
```

**Inventory**
```
location       id, business_id, name, type, address, geo_area_id

stock_movement id, business_id, variant_id, location_id,
               type(in|out|adjust|transfer|reserve|release),
               qty, reason, ref_type, ref_id, created_at, created_by

stock_level    business_id, variant_id, location_id,
               on_hand, reserved, updated_at         (cache, rebuildable)
```

**Orders**
```
order          id, business_id, number, contact_id, status, source,
               subtotal_cents, discount_cents, shipping_cents, total_cents,
               currency, assigned_to, notes, custom_fields(jsonb),
               idempotency_key, created_at

order_line     id, order_id, variant_id,
               name_snapshot, price_snapshot_cents, cost_snapshot_cents,
               qty, line_total_cents

order_event    id, order_id, from_status, to_status,
               actor_type(user|ai|system), actor_id, reason, created_at
```

**Payments**
```
payment        id, business_id, order_id, method(cod|link|instapay|cash),
               amount_cents, currency,
               status(pending|paid|failed|settled), provider_ref, paid_at

refund         id, payment_id, amount_cents, reason, created_at

invoice        id, business_id, order_id, number, issued_at, pdf_url
```

**Shipping**
```
shipment       id, business_id, order_id, carrier, tracking_number,
               status, cod_amount_cents, fee_cents, created_at

tracking_event id, shipment_id, status, description, occurred_at

zone           id, business_id, name, geo_area_ids[], fee_cents, eta_days

cod_settlement id, business_id, carrier, amount_cents, fee_cents,
               settled_at, matched_shipment_ids[]
```

**Finance**
```
expense        id, business_id, category, amount_cents, currency,
               date, note, receipt_url

ledger_entry   id, business_id, account, debit_cents, credit_cents,
               ref_type, ref_id, occurred_at
```

### 3.3 Order state machine

```
draft ---> confirmed ---> packed ---> shipped ---> delivered
  |            |             |           |
  |            v             v           v
  +------> cancelled     cancelled    returned
```

| Transition | Side effects |
|---|---|
| draft to confirmed | reserve stock, emit `OrderCreated` |
| confirmed to packed | none |
| packed to shipped | stock out movement, create shipment, emit `OrderShipped` |
| shipped to delivered | COD payment marked paid, emit `OrderDelivered` |
| any to cancelled | release reserved stock, emit `OrderCancelled` |
| delivered to returned | stock in movement, create refund, emit `OrderReturned` |

Transitions live in one file. Any transition not listed is rejected.

### 3.4 What ERP exposes

**Emits:** `OrderCreated`, `OrderShipped`, `OrderDelivered`, `OrderCancelled`, `OrderReturned`, `PaymentReceived`, `StockLow`

**Read-only service methods for other modules:**
`getOrdersByContact(contactId)`, `getOrder360(number)`, `getCustomerSpend(contactId)`

**AI tools:** `check_stock`, `get_price`, `get_shipping_fee`, `create_order`, `get_order_360`, `search_orders`, `get_metric`

**Never does:** write to CRM or Channels tables, or send messages. It emits an event and Channels decides what to send.

### 3.5 ERP rules

1. All writes go through services, never straight to tables. The AI and the UI use the same services
2. Stock is a ledger. Never edit a quantity, add a movement. Levels are a rebuildable cache
3. Money as integers (`_cents`) plus a currency field. Never floats
4. Per-business numbering (#1042), not UUIDs, for anything a user reads
5. Idempotency keys on order creation. The AI will retry
6. Snapshot name, price and cost on order lines. Catalog changes must not rewrite history
7. Unit tests on every money path: totals, discounts, refunds, partial payments

### 3.6 ERP build steps

| Step | Days | What |
|---|---|---|
| 1 | 2 | State machine and money rules on paper, before any code |
| 2 | 3 | Schema for catalog and inventory, RLS policies, tenant isolation test |
| 3 | 5 | Catalog CRUD and first real API, shared Zod schemas |
| 4 | 3 | Stock movements, cached level, rebuild function |
| 5 | 8 | Orders, state machine, reserve and release, outbox events, numbering |
| 6 | 4 | Payments, COD as a method that settles later |
| 7 | 5 | Shipping, one carrier adapter, tracking webhook, COD reconciliation |
| 8 | 3 | Expenses and profit |

---

## 4. CRM

### 4.1 Code structure

```
apps/api/src/modules/crm/
├── contacts/
│   ├── contact.entity.ts
│   ├── organization.entity.ts
│   ├── contact-identity.entity.ts
│   ├── contact-merge.entity.ts
│   ├── contact.service.ts           (findOrCreate, update, assign)
│   ├── identity.service.ts          (normalize, match, merge)
│   ├── phone-normalizer.ts          (Egyptian numbers to E.164)
│   └── contact.controller.ts
├── timeline/
│   ├── activity.entity.ts
│   ├── activity.listener.ts         (consumes ERP + Channels events)
│   └── timeline.service.ts          (read only)
├── stats/
│   ├── contact-stats.entity.ts
│   ├── stats.listener.ts
│   └── stats.service.ts
├── segments/
│   ├── segment.entity.ts
│   ├── segment-evaluator.ts         (filter json to query)
│   └── segment.service.ts
├── pipelines/
│   ├── pipeline.entity.ts
│   ├── stage.entity.ts
│   ├── deal.entity.ts
│   ├── deal-event.entity.ts
│   └── pipeline.service.ts
├── engagement/
│   ├── tag.entity.ts
│   ├── note.entity.ts
│   ├── task.entity.ts
│   └── engagement.service.ts
├── consent/
│   ├── consent.entity.ts
│   └── consent.service.ts
├── crm.events.ts
├── crm.tools.ts
└── crm.module.ts
```

### 4.2 Data model

**Contacts and identities**
```
contact          id, business_id, name, organization_id, owner_user_id,
                 geo_area_id, lead_source, lifecycle_stage,
                 custom_fields(jsonb), created_at, first_seen_at

organization     id, business_id, name, industry, size,
                 owner_user_id, custom_fields(jsonb)

contact_identity id, business_id, contact_id,
                 type(phone|whatsapp|instagram|email),
                 value_normalized, value_raw, verified, created_at
                 UNIQUE (business_id, type, value_normalized)

contact_merge    id, business_id, kept_contact_id, merged_contact_id,
                 merged_payload(jsonb), merged_by, merged_at
```

**Timeline**
```
activity         id, business_id, contact_id,
                 type(message|order|payment|note|task|stage_change|call),
                 source_module, ref_type, ref_id, summary,
                 actor_type(user|ai|system), actor_id, occurred_at
                 INDEX (business_id, contact_id, occurred_at desc)
```

**Stats (cache, rebuildable)**
```
contact_stats    business_id, contact_id,
                 orders_count, total_spent_cents, avg_order_cents,
                 first_order_at, last_order_at, last_message_at,
                 open_deals_count, updated_at
```

**Segments**
```
segment          id, business_id, name, filter(jsonb), is_dynamic, created_by
segment_member   segment_id, contact_id, added_at     (static segments only)
```

**Pipelines and deals**
```
pipeline         id, business_id, name, object_type, from_template
stage            id, pipeline_id, name, position, probability, is_won, is_lost
deal             id, business_id, pipeline_id, stage_id, contact_id,
                 organization_id, title, value_cents, currency,
                 owner_user_id, expected_close_date,
                 lost_reason, closed_at, custom_fields(jsonb)
deal_event       id, deal_id, from_stage_id, to_stage_id, actor_id, created_at
```

**Engagement and consent**
```
tag              id, business_id, name, color
contact_tag      contact_id, tag_id
note             id, business_id, contact_id, body, created_by, created_at
task             id, business_id, contact_id, deal_id, title,
                 assigned_to, due_at, done_at, created_by
consent          id, business_id, contact_id, channel,
                 status(opted_in|opted_out), source, occurred_at
```

### 4.3 Identity matching flow

The piece most likely to go wrong, so it gets explicit rules.

```
incoming message: whatsapp_id 201234567890
        |
        v
normalize value  ->  +201234567890
        |
        v
lookup contact_identity (business_id, type, value_normalized)
        |
   found? --yes--> return existing contact        (idempotent)
        |
        no
        v
create contact + identity  --> emit ContactCreated
        |
        v
look for possible duplicates (same digits on another identity type,
same name and area)  --> create a merge SUGGESTION, never auto-merge
```

**Egyptian phone normalization:** strip spaces and dashes, convert leading `0` to `+20`, handle `002` prefix, handle `01x` mobile prefixes, store both normalized and raw.

**Why suggest and not auto-merge:** shared family phones and people with several numbers make auto-merge dangerous and hard to undo.

### 4.4 What CRM consumes and emits

**Consumes:**

| Event from | Effect in CRM |
|---|---|
| `MessageReceived` (Channels) | find or create contact, link conversation, add activity, update `last_message_at` |
| `OrderCreated` (ERP) | activity, stats update, move deal stage, set lifecycle to customer |
| `PaymentReceived` (ERP) | activity, stats update |
| `OrderDelivered` (ERP) | activity, schedule follow-up task |

**Emits:** `ContactCreated`, `ContactMerged`, `DealStageChanged`, `DealWon`, `DealLost`, `TaskAssigned`, `ConsentChanged`

**Read-only service methods:**
`getContact(id)`, `findContactByIdentity(type, value)`, `getSegmentContacts(segmentId)`, `getContactStats(id)`

**AI tools:** `find_customer`, `get_customer_360`, `get_customer_history`, `create_note`, `create_task`, `search_contacts`, `get_segment`, `update_deal_stage`

### 4.5 CRM rules

1. CRM never writes orders, payments or messages. It reacts to events and reads by service call
2. Contact creation is idempotent, keyed on the normalized identity
3. Merges are suggested, never automatic, and always logged so they can be undone
4. Activity is append-only, written only by listeners
5. Stats are a cache with a rebuild function that recomputes from source
6. Segments are stored as filter JSON, evaluated at read time

### 4.6 CRM build order

1. Contacts, identities and phone normalization (first, everything hangs off it)
2. Timeline consuming ERP and Channels events
3. Stats from events
4. Tags, notes, tasks
5. Segments
6. Pipelines and deals
7. Consent

---

## 5. Channels

Separate from CRM because they change for different reasons. Channels is integration-heavy (webhooks, Meta rules, retries). CRM is about people and should not care which channel a message came from. Mixed together, every new channel means touching CRM code.

### 5.1 Code structure

```
apps/api/src/modules/channels/
├── accounts/
│   ├── channel-account.entity.ts
│   └── account.service.ts
├── conversations/
│   ├── conversation.entity.ts
│   ├── message.entity.ts
│   ├── attachment.entity.ts
│   ├── conversation.service.ts
│   └── inbox.service.ts             (state, assignment)
├── adapters/
│   ├── channel-adapter.interface.ts
│   ├── whatsapp/
│   │   ├── whatsapp.adapter.ts
│   │   ├── whatsapp.webhook.ts
│   │   ├── template.service.ts      (24-hour window handling)
│   │   └── whatsapp.mapper.ts       (to normalized Message)
│   ├── instagram/
│   └── store/
├── media/
│   ├── transcription.service.ts     (voice notes)
│   └── storage.service.ts
├── channels.events.ts
└── channels.module.ts
```

### 5.2 Data model

```
channel_account  id, business_id, type(whatsapp|instagram|store),
                 external_id, phone_number, access_token_ref,
                 status, connected_at

conversation     id, business_id, channel_account_id, contact_id,
                 external_thread_id, state(open|waiting|closed),
                 assigned_to(user_id|ai), last_message_at,
                 window_expires_at

message          id, business_id, conversation_id,
                 direction(in|out), external_id,
                 body, type(text|image|audio|location|template),
                 sender_type(customer|ai|user), sender_id,
                 delivery_status, created_at
                 UNIQUE (business_id, external_id)     (idempotency)

attachment       id, message_id, type, url, mime, transcript
```

### 5.3 Requirements

- One normalized `Message` model, one adapter per channel
- Webhook, then queue, then process, with **idempotency**: Meta sends duplicates
- **24-hour window:** outside it only approved template messages can be sent. Track `window_expires_at` per conversation
- **Voice notes:** a large share of MENA WhatsApp messages are voice. Transcription must handle Egyptian dialect, not just standard Arabic
- Images: customers send photos of products and receipts
- Rate limits and retries per channel
- Delivery and read status tracked per message

### 5.4 What Channels exposes

**Emits:** `MessageReceived`, `MessageSent`, `MessageFailed`, `ConversationAssigned`, `WindowExpiring`

**Consumes:** `OrderCreated` (send confirmation), `OrderShipped` (send tracking), `OrderDelivered` (follow-up)

**AI tools:** `send_message`, `handoff_to_human`, `get_conversation_history`

---

## 6. AI layer

Two agents on one engine, different tools and permissions.

### 6.1 Code structure

```
apps/ai/
├── agents/
│   ├── customer_agent.py
│   ├── owner_copilot.py
│   └── base_agent.py
├── brain/
│   ├── business_brain.py            (profile + decisions + memory)
│   ├── rag.py                       (catalog, FAQs, policies)
│   └── embeddings.py
├── tools/
│   ├── registry.py                  (filters by capabilities + role)
│   ├── erp_tools.py
│   ├── crm_tools.py
│   └── channel_tools.py
├── guardrails/
│   ├── confidence.py
│   ├── approval_queue.py
│   └── pii_masking.py
├── metrics/
│   └── metric_layer.py              (predefined metrics, no free SQL)
├── eval/
│   ├── dataset.jsonl                (150+ real messages, ar/franco/en)
│   └── run_eval.py
└── main.py                          (FastAPI)
```

### 6.2 Customer agent

- Answers buyers in Egyptian Arabic, Franco and English
- Voice notes transcribed first
- Takes orders and bookings
- Confirms COD, sends tracking
- Hands off when unsure
- Very limited tool set

### 6.3 Owner copilot

- "What about order 144", "who owes me money", "how much did I sell in Alex last month"
- Adds products, creates campaigns
- Builds fields and automations on request, shown for confirmation
- Answers with cards and actions, not walls of text
- Limited by the user's role: a staff member asking for profit is refused, because the tools check the role

### 6.4 Business brain

"Tuned to their business" does **not** mean fine-tuning a model per business. That is expensive, slow, and stale the moment a price changes. It comes from three things:

1. **Business brain:** a living profile (what they sell, prices, tone, goals, past decisions) read by every AI feature
2. **Tools:** live access to real data
3. **Memory:** what the owner decided before ("we stopped selling size S", "focus on Cairo this month")

Change a price and the AI knows instantly.

### 6.5 Rules

1. **Exact data comes from tools, never from RAG.** RAG is only for fuzzy text (FAQs, policies)
2. **360 tools for complete answers.** `get_order_360(144)` returns order, customer, items, payment status, tracking and timeline in one call
3. **A metrics layer instead of free SQL.** The agent picks from predefined metrics and filters. Read-only SQL on views is a later fallback
4. **Tools follow capabilities and roles.** If inventory is off, `check_stock` does not exist for that business
5. **Risky actions need approval.** Spending money, bulk messages: the AI drafts, the owner confirms, everything logged
6. **Handle ambiguity and follow-ups.** Three customers named Mariam means asking which one. "And did she pay?" remembers who
7. **Answer with cards.** Short summary plus a card with buttons
8. **Measured, not claimed.** 150+ real messages as an eval set, rerun on every change

### 6.6 Tool contracts (starting set)

```
get_order_360(number)
find_customer(name_or_phone)
get_customer_360(contact_id)
get_payments(customer_or_order)
search_orders(status, date_range, area, assigned_to)
get_metric(metric, period, group_by, filters)
check_stock(variant_id)
get_price(variant_id)
get_shipping_fee(geo_area_id)
create_order(contact, lines, payment_method, address)
create_note(contact_id, body)
create_task(contact_id, title, assigned_to, due_at)
send_message(conversation_id, body)
handoff_to_human(conversation_id, reason)
```

### 6.7 Example answer format

```
Order 144, Mariam Ahmed
Black hoodie L, 700 EGP, cash on delivery
Shipped yesterday, expected tomorrow
Payment not collected yet
[Track shipment] [Message Mariam] [Mark as paid]
```

The copilot returns data plus a component name. The frontend renders it with the same component used on the order page. One component library for the app and the AI.

### 6.8 Model strategy

**Production:** an API model (Claude, GPT, Gemini). Reliable multi-step tool calling is the whole product, and small self-hosted models are weak at it. Test all three on the eval set and pick by results, not hype.

Routing: a small cheap model for simple intent classification, a strong model for reasoning and tool use.

**Research chapter:** LoRA or QLoRA fine-tune of a small open model (Qwen 2.5 7B or 14B, or an Arabic-tuned model) on Egyptian Arabic commerce dialogue. Run it on the same eval set and compare against the APIs on dialect accuracy, tool-call correctness, latency and cost.

Why not self-host in production: a 70B-class model needs around 2x A100 80GB or 1x H100, roughly $1,000 to $2,000 per month, idle most of the time for a pilot of 5 businesses. API calls for the whole project cost tens of dollars.

Dataset for the fine-tune: 1,000 to 5,000 Egyptian Arabic business chat examples. Generate synthetic ones with a strong API model, then clean by hand. The dataset is the real bottleneck, not the training.

---

## 7. Marketing

```
apps/api/src/modules/marketing/
├── campaigns/
│   ├── campaign.entity.ts           (segment + message + schedule)
│   ├── campaign-run.entity.ts
│   └── campaign.service.ts
├── content/
│   ├── content-item.entity.ts       (post, caption, draft)
│   ├── calendar.service.ts
│   └── publisher.service.ts         (Meta Graph API)
├── followups/
│   ├── followup-rule.entity.ts
│   └── followup.listener.ts         (reacts to OrderDelivered etc)
├── geo/
│   └── schema-generator.ts          (product schema, FAQs for AI search)
└── marketing.module.ts
```

- Campaign = segment plus message plus schedule
- Consent is checked before every send, for every contact
- GEO means generating structured data, FAQs and clean descriptions on the store page so AI search tools can understand and recommend the business
- Ads API is a stretch item, ad copy suggestions come first

---

## 8. Brand studio

```
apps/api/src/modules/brand/
├── brand-kit/
│   ├── brand-kit.entity.ts          (colors, fonts, logo config, tone)
│   ├── logo-composer.ts             (SVG templates + fonts + colors)
│   └── brand.service.ts
├── store-page/
│   ├── site-config.entity.ts        (JSON the theme renders)
│   ├── theme-registry.ts
│   └── publisher.service.ts
└── brand.module.ts
```

- **Do not generate logo images with a model.** Results are inconsistent. Compose from SVG templates plus fonts plus AI-picked colors
- Store page is a few Next.js themes rendered from JSON the AI fills
- Not a drag-and-drop builder. That is a project on its own

---

## 9. Founder studio

```
apps/api/src/modules/founder/
├── validation/                      (grounded market research, competitors)
├── model/                           (business model canvas, pricing)
├── projection/                      (simple financial projection)
├── deck/                            (pitch deck from structured JSON)
└── legal/                           (Egypt checklist: commercial register,
                                      tax card, e-invoicing)
```

Every output is structured JSON the AI fills, rendered by your own templates. Market research grounded with web search so it is not invented.

Mostly designed, not built, in the project scope.

---

## 10. Reports

```
apps/api/src/modules/reports/
├── summaries/
│   ├── daily-summary.entity.ts
│   ├── product-summary.entity.ts
│   └── summary.listener.ts          (consumes ERP + CRM events)
├── metrics/
│   └── metric-definitions.ts        (the same layer the AI uses)
└── reports.module.ts
```

- Event consumers maintain summary tables
- The KPI set comes from the template pack
- Feeds Home and the copilot's `get_metric` tool
- Read only, never writes business data

---

## 11. Frontend structure

```
apps/web/src/
├── app/
│   ├── (dashboard)/
│   │   ├── home/
│   │   ├── inbox/
│   │   ├── orders/
│   │   ├── products/
│   │   ├── customers/
│   │   ├── marketing/
│   │   ├── store/
│   │   └── settings/
│   ├── copilot/
│   ├── onboarding/
│   └── s/[slug]/                    (generated store pages)
├── components/
│   ├── records/                     (the Lightning-style component set)
│   │   ├── RecordHeader.tsx
│   │   ├── StatusPath.tsx
│   │   ├── LineItems.tsx
│   │   ├── CustomerCard.tsx
│   │   ├── ShipmentCard.tsx
│   │   ├── PaymentCard.tsx
│   │   ├── Timeline.tsx
│   │   └── registry.ts              (name to component map)
│   ├── data/
│   │   ├── DataTable.tsx
│   │   ├── KanbanBoard.tsx
│   │   └── SavedViews.tsx
│   ├── fields/                      (generated from the field registry)
│   └── ai/
│       ├── CopilotPanel.tsx
│       └── AiCard.tsx               (renders what the copilot returns)
└── lib/
    ├── layout-renderer.tsx          (layout JSON to components)
    └── capabilities.ts
```

### Layout rendering

```json
{
  "object": "order",
  "template": "clothing_brand",
  "layout": [
    { "component": "RecordHeader" },
    { "component": "StatusPath" },
    { "component": "LineItems", "props": { "showVariants": true } },
    { "component": "ShipmentCard" },
    { "component": "CustomerCard" },
    { "component": "Timeline" }
  ]
}
```

A service business swaps `LineItems` for `BookingSlot`. Same code, different JSON.

RTL and mobile from day one, not retrofitted.

---

## 12. Cross-cutting: what a team of 5 to 10 needs

Lives in Core, but without it the product does not work for a team.

1. **Roles and permissions:** owner, manager, sales, support, viewer. Who sees profit, who can discount
2. **Assignment and ownership:** on orders, contacts, deals, conversations
3. **Tasks and mentions:** "@Mariam follow up with this client"
4. **Audit log:** who changed the price, who cancelled the order. Not optional with a team
5. **Notifications:** assigned to you, deal stuck, stock low
6. **Saved views per user:** "my open deals", "orders to ship today"

---

## 13. Stack

| Layer | Choice | Why |
|---|---|---|
| Repo | Turborepo monorepo | Shared types across backend, frontend and AI service |
| Backend | NestJS, one module per section above | Structure enforced by the framework |
| Database | PostgreSQL + pgvector, row-level security | Tenant isolation enforced by the database, not by discipline |
| ORM | Drizzle | Typed, works cleanly with RLS and raw SQL |
| Queue | Redis + BullMQ | Webhooks, retries, scheduled follow-ups |
| Frontend | Next.js, Tailwind, shadcn/ui, TanStack Table, React Hook Form | RTL and mobile first |
| AI service | Python FastAPI, tool calling | Separate so models can be swapped and evaluated |
| Validation | Shared Zod schemas | One source of truth for types and forms |

### Hosting

- One VPS (8GB RAM, around 10 EUR/month, Hetzner CX32 or similar): backend, Postgres, Redis, AI service. Enough for a pilot of 5 businesses
- Frontend on Vercel free tier
- LLM, speech to text and embeddings via API
- Rented hourly GPU (RunPod, Vast.ai) only for the fine-tuning experiment

### Global data rules

1. Money as integers (`_cents`) plus currency. Never floats
2. Stock is a ledger, levels are a rebuildable cache
3. Per-business numbering for anything a user reads
4. Adapters for the outside world: shipping, payments, channels behind interfaces
5. Outbox table for reliable events
6. Audit log on everything
7. Automated tests that try to read another business's data and must fail

---

## 14. Build order

```
Core
 -> ERP lite
 -> CRM
 -> Channels (WhatsApp)
 -> AI
 -> Brand studio
 -> Marketing
 -> Reports
```

Why this order: ERP before CRM because orders and money are the value, and CRM mostly consumes ERP events. Channels after both so there is something real to attach chats to. AI last of the core five, because an agent with no tools is a chatbot and you would build it twice.

**Running in parallel from week one, neither blocks code:**
- Meta business verification and WhatsApp Cloud API setup
- Collecting real Egyptian Arabic messages for the eval set and the fine-tuning dataset

---

## 15. What to study, not fork

- **Chatwoot:** omnichannel inbox design
- **Twenty:** modern open-source CRM
- **ERPNext:** ERP data models
- **Medusa:** commerce architecture
- **Salesforce:** its object model (Account, Contact, Lead, Opportunity, Activity), Agentforce's shape (topics, actions, guardrails, audit log), and the Einstein Trust Layer idea of masking personal data before sending to the LLM and logging every AI action
- What **not** to take from Salesforce: the configure-everything approach. Users should not design pages
