# Technical Structure

> The design of the whole product: what it is, how it is built, what each part
> owns, and how the AI manages it. A living document and the single source of
> truth. When the team changes something, this file changes first.
>
> It says **what** the system is, not which tools build it. The stack is still
> open (see section 13).

---

## 0. The idea in one page

**One AI for every small business.** Each business gets its own AI manager that
knows only that business. The AI is the main part of the product: it talks to
customers, takes orders, watches stock and money, and tells the owner what needs
attention. Behind it sit a full ERP, a CRM, message channels, a store website and
the other tools. The owner never sees those words; they see an AI partner that
runs the daily work with them.

**Showcase category:** small local businesses, starting with **coffee shops**
and **small markets** (groceries, mini-markets). The engine is built for any
category; these two prove it.

**The model:** our own model, an open base model that we fine-tune to manage a
small business in Egyptian Arabic and English through tools. It adapts to any
category. No ready-made commercial model (GPT, Claude, Gemini) is used anywhere.

```
+--------------------------------------------------------------------+
|  AI layer: AI manager (one engine, our tuned model)                |
|  customer agent | owner partner | proactive jobs | business brain  |
+---------------------------+----------------------------------------+
                            | tools only, never direct data access
+---------+---------+------------+----------+----------+------------+
|  ERP    |  CRM    |  Channels  |  Store   |  Studio  | Marketing  |
| menu    | customers| website   | website  | brand    | offers     |
| recipes | loyalty | chat       | menu,    | menu     | campaigns  |
| stock   | timeline| Telegram   | ordering | design   | follow-ups |
| sales   | segments| WhatsApp   |          | posts    |            |
| shifts  |         | Instagram  |          |          |            |
| supplier|         |            |          |          |            |
+---------+---------+------------+----------+----------+------------+
+--------------------------------------------------------------------+
|  Reports: numbers built from events                                |
+--------------------------------------------------------------------+
|  Core: businesses, members, roles, templates, audit, events, one DB |
+--------------------------------------------------------------------+
```

---

## 1. The problem and the people

### 1.1 Coffee shops

1. They do not know the real cost of a cup. Beans, milk and sugar prices change
   often, so items quietly lose money
2. Waste nobody tracks: spoiled milk, unsold pastries, over-pouring, free drinks
3. Cash does not match sales at the end of a shift, and nobody knows why
4. Stock runs out at the busiest hour because nobody reordered in time
5. Delivery apps take a large cut and keep the customer
6. Chat orders (WhatsApp, Instagram) are missed or not ready on time
7. Regulars are not remembered; loyalty is a paper card or nothing
8. Strong competition; offers are guesses with no measurement
9. The owner is not always there and cannot see what happens
10. Group bookings (match nights, events) live in chats and paper

### 1.2 Small markets

1. The credit notebook: neighbours buy on account and debts are forgotten
2. Many products, frequent supplier price changes, prices go out of date
3. Expiry dates (dairy and similar) found too late
4. Neighbourhood delivery orders by phone and WhatsApp, written on paper
5. Cash does not match sales; shortages and theft go unnoticed
6. Reordering from many suppliers by memory

### 1.3 Who uses the product

| Who | Uses | Through |
|-----|------|---------|
| Owner | Everything their role allows; talks to the AI partner | Dashboard, phone, chat with the AI |
| Staff (barista, cashier, manager) | Sales screen, orders, shift close, waste, what their role allows | Sales screen, dashboard |
| Customers (buyers) | Ask questions, order, book, collect loyalty | The business's chat channels and store website |
| Suppliers | Receive purchase orders (term 2: confirm them) | Messages from the system |

### 1.4 One journey, one product

Idea, validate, build, launch, run, grow. A coffee shop that already exists
starts at **run**. A new one starts earlier (brand, menu, store page from the
studio). The deepest part of the project is **run and grow**: the daily work the
AI takes over.

### 1.5 Simple outside

The owner never sees "ERP" or "CRM". Six principles:

1. **Talk, not forms.** "Add iced latte, 75 pounds" or a voice note does it
2. **Nothing typed twice.** Customers and orders come straight from chats
3. **Templates by business type.** Pick "coffee shop" and it is ready
4. **Features appear when needed.** Branches only show when there is a second one
5. **Home is a to-do list.** "Milk runs out tomorrow, 2 chats need you, cash was
   50 pounds short yesterday", not charts
6. **Setup in minutes.** A photo of the menu becomes products and prices

Test for every screen: could a café owner use it without help?

### 1.6 Why owners keep using it

1. **Visible money.** A monthly AI report: waste saved, delivery fees avoided,
   cash gaps caught, all in pounds
2. **Used every hour.** The sales screen is the café's register, not an extra tool
3. **Their customers depend on it.** Orders, chats and loyalty points live here
4. **Their history lives here.** Recipes, costs, suppliers, customers, sales
5. **The AI learns them.** Memory of decisions makes it better every month
6. **Hard to break.** Fast at rush hour, keeps selling when the internet drops

---

## 2. Architecture rules

One backend made of strict modules and one database. Not microservices.

1. **One owner per piece of data.** Others reference it by ID, never copy it
2. **Modules talk through events.** No module touches another module's tables
3. **Reading across modules is a service call**
4. **The AI acts only through tools.** It owns no data and never reads the database
5. **Every write is audited, every state change emits an event**

### 2.1 Data rules

- Money as whole numbers in piastres (`_cents` style) plus a currency. Never floats
- Stock is a ledger of movements. Levels are a cache that can be rebuilt
- Numbers people read are per business (order #1042, purchase #33), not random IDs
- Idempotency keys on anything that can be retried (orders, webhooks, AI actions)
- Order lines snapshot name, price and cost at the time of sale
- Every business-scoped table has `business_id`, locked by row-level security
- Events go through an outbox, written in the same transaction as the change
- Outside services (channels, payments, couriers) sit behind adapters
- A test that tries to read another business's data, and must fail

### 2.2 Who owns what

| Data | Owner |
|------|-------|
| Businesses, members, roles, templates, audit | Core |
| Items, recipes, stock, sales, orders, shifts, suppliers, money | ERP |
| Customers, identities, timeline, loyalty, segments | CRM |
| Channel accounts, conversations, messages | Channels |
| Store website settings and pages | Store website |
| Brand kit, designs, posts | Studio |
| Offers, campaigns | Marketing |
| Summary numbers | Reports |
| Business brain, AI memory, approvals, AI action log | AI layer (its own data only; business data only through tools) |

---

## 3. Core

The foundation every module stands on.

### 3.1 What it owns

```
business          id, name, slug, business_type, currency, timezone, locale,
                  capabilities(jsonb), settings(jsonb), created_at
user              id, email, phone, name, password_hash, locale
membership        id, business_id, user_id, role, invited_by, joined_at
                  UNIQUE (business_id, user_id)
role              business_id, name, permissions(jsonb)
                  defaults: owner, manager, staff, viewer
location          id, business_id, name, address, area      (branches)
template_pack     key, version, content(jsonb)              (files, versioned)
audit_log         id, business_id, actor_type(user|ai|system), actor_id,
                  action, object_type, object_id, before, after, created_at
outbox            id, business_id, event_type, payload, created_at,
                  dispatched_at, attempts, last_error
notification      id, business_id, user_id, type, title, body,
                  ref_type, ref_id, read_at, created_at
```

### 3.2 Templates: one engine, any business

A business's app is composed from configuration, not code. A template pack holds:

| Part | Coffee shop | Small market |
|------|-------------|--------------|
| Capabilities | recipes, modifiers, tables, loyalty, pickup and delivery | credit accounts, expiry dates, barcodes, delivery |
| Item fields | size, milk type, extra shot | brand, unit, barcode, expiry |
| Order stages | new, preparing, ready, collected or delivered | new, packed, out for delivery, delivered |
| Home items | stock running out, cash gap, slow hours | debts due, items expiring, stock running out |
| AI rules | tone, which actions need approval | tone, credit limits, which actions need approval |

`capabilities` = template defaults merged with the business's own overrides. The
screens, the AI tools and the event listeners all read the same result. If a
capability is off, its screens hide and its AI tools do not exist for that business.

A new business type is a new template file, not new code.

### 3.3 Events infrastructure

1. A service writes the change and an outbox row in one transaction
2. A dispatcher reads undispatched rows and delivers them to listeners
3. Delivered rows are marked; failures retry

---

## 4. ERP

Operations: what the business sells and what happens to it.

### 4.1 Catalog and recipes

```
item            id, business_id, name, category_id, type(product|menu_item|service),
                custom_fields(jsonb), active
variant         id, item_id, name, sku, barcode, attributes(jsonb),
                price_cents, currency, active
modifier_group  id, business_id, name (milk type, extra shot), rules(jsonb)
modifier        id, group_id, name, price_cents, ingredient_effects(jsonb)
category        id, business_id, name, parent_id
ingredient      id, business_id, name, unit (g, ml, piece), cost_cents_per_unit
recipe_line     variant_id, ingredient_id, qty             (what one sale uses)
```

A sale of a menu item deducts its recipe's ingredients. A product sold as-is
(a bottle of water, a pack of biscuits) deducts itself.

### 4.2 Stock

```
stock_movement  id, business_id, location_id, stock_ref(ingredient|variant),
                type(in|out|sale|waste|adjust|transfer|reserve|release),
                qty, unit_cost_cents, reason, expiry_date, ref_type, ref_id,
                created_at, created_by
stock_level     business_id, location_id, stock_ref, on_hand, reserved   (cache)
```

Waste is a movement with a reason ("spoiled milk"), logged by tapping or by
telling the AI.

### 4.3 Sales and orders

```
order           id, business_id, number, location_id, customer_id, channel
                (counter|website|chat|phone), type(dine_in|pickup|delivery),
                status, subtotal_cents, discount_cents, delivery_cents,
                total_cents, currency, shift_id, created_by_type(user|ai),
                idempotency_key, notes, created_at
order_line      id, order_id, variant_id, modifiers(jsonb),
                name_snapshot, price_snapshot_cents, cost_snapshot_cents,
                qty, line_total_cents
order_event     id, order_id, from_status, to_status, actor_type, actor_id,
                reason, created_at
payment         id, business_id, order_id, method(cash|card|wallet|on_account),
                amount_cents, status, paid_at
```

Order life (stages come from the template):

```
new --> accepted --> preparing --> ready --> collected / delivered
 |         |            |
 +---------+------------+--> cancelled
```

| Move | What happens |
|------|-------------|
| new to accepted | reserve stock, emit `OrderCreated` |
| accepted to preparing | none |
| ready | notify the customer on their channel |
| collected or delivered | deduct stock (sale movement), mark paid if cash, emit `OrderCompleted` |
| cancelled | release reserved stock, emit `OrderCancelled` |

A counter sale is created and completed in one step. All moves live in one
place; any other move is rejected.

### 4.4 Shifts and cash

```
shift           id, business_id, location_id, opened_by, opened_at,
                opening_cash_cents, closed_by, closed_at, counted_cash_cents,
                expected_cash_cents, difference_cents, note
```

Closing a shift compares counted cash with expected cash from sales. A gap emits
`CashGapFound`.

### 4.5 Suppliers and purchases

```
supplier        id, business_id, name, phone, items_supplied(jsonb)
purchase_order  id, business_id, number, supplier_id, status
                (draft|approved|sent|received|cancelled), total_cents, created_by_type
purchase_line   id, purchase_order_id, stock_ref, qty, unit_cost_cents
```

Receiving a purchase adds stock movements and updates ingredient costs. A cost
change emits `CostChanged`, which can trigger a price suggestion.

### 4.6 Credit accounts (small markets)

```
credit_account  id, business_id, customer_id, limit_cents, balance_cents
credit_entry    id, account_id, type(charge|payment), amount_cents, ref, created_at
```

### 4.7 Money

```
expense         id, business_id, category, amount_cents, date, note, receipt_url
```

Profit per item = price minus recipe cost at the time of sale (from the snapshot).

### 4.8 What ERP exposes

**Emits:** `OrderCreated`, `OrderReady`, `OrderCompleted`, `OrderCancelled`,
`PaymentReceived`, `StockLow`, `StockExpiring`, `WasteLogged`, `CashGapFound`,
`CostChanged`, `PurchaseReceived`

**Read calls:** `getOrder360(number)`, `getOrdersByCustomer(id)`, `getItemCost(variant)`,
`getStockLevels(filter)`, `getSales(period, group_by)`

**Never does:** write to CRM or Channels, or send messages. It emits events.

---

## 5. CRM

Relationships: who the customers are.

```
customer         id, business_id, name, area, lifecycle_stage, owner_user_id,
                 custom_fields(jsonb), first_seen_at
customer_identity id, business_id, customer_id, type(phone|whatsapp|telegram|
                 instagram|email), value_normalized, value_raw, verified
                 UNIQUE (business_id, type, value_normalized)
activity         id, business_id, customer_id, type(message|order|payment|note|
                 task|loyalty|booking), source_module, ref_type, ref_id,
                 summary, actor_type, occurred_at
customer_stats   customer_id, orders_count, total_spent_cents, avg_order_cents,
                 usual_order(jsonb), last_order_at, last_message_at      (cache)
loyalty_account  customer_id, points, tier
loyalty_entry    id, customer_id, points, reason, ref, created_at
booking          id, business_id, customer_id, starts_at, party_size, deposit_cents,
                 status, note
tag, customer_tag, note, task                                  (engagement)
segment          id, business_id, name, filter(jsonb)          (term 2)
consent          id, business_id, customer_id, channel, status (term 2)
```

### 5.1 Identity matching

A message from a phone number is normalized (Egyptian numbers to +20 form) and
looked up. Found: same customer. Not found: new customer. Possible duplicates
become a merge **suggestion**, never an automatic merge.

### 5.2 What CRM consumes and emits

| Event | Effect in CRM |
|-------|---------------|
| `MessageReceived` | find or create the customer, add activity |
| `OrderCreated`, `OrderCompleted` | activity, stats, usual order, loyalty points |
| `PaymentReceived` | activity, credit balance if on account |

**Emits:** `CustomerCreated`, `CustomerInactive` (a regular stopped coming),
`LoyaltyRewardReached`, `BookingCreated`

---

## 6. Channels

Where messages come from. Separate from CRM so a new channel never touches CRM.

```
channel_account  id, business_id, type(website_chat|telegram|whatsapp|instagram),
                 external_id, token_ref, status, connected_at
conversation     id, business_id, channel_account_id, customer_id,
                 external_thread_id, state(open|waiting|closed),
                 handled_by(ai|user), assigned_to, last_message_at, window_expires_at
message          id, business_id, conversation_id, direction(in|out), external_id,
                 body, type(text|image|audio|location|template), sender_type
                 (customer|ai|user), delivery_status, created_at
                 UNIQUE (business_id, external_id)
attachment       id, message_id, type, url, transcript
```

- One normalized message shape, one adapter per channel
- Webhook, then queue, then process; duplicates are ignored
- **Order of channels:** website chat and Telegram first (no outside approval),
  WhatsApp as soon as Meta approves (start the request in week 4), Instagram later
- Voice notes are transcribed; images are kept for the AI (menu photos, receipts)

**Emits:** `MessageReceived`, `MessageSent`, `ConversationHandedOff`
**Consumes:** `OrderReady` (tell the customer), `OrderCreated` (confirmation)

---

## 7. Store website

Each business gets a simple website it manages from the dashboard:

- Menu or catalog from the ERP, always live (prices, what is available)
- Ordering for pickup or delivery, with payment on collection or delivery
- A chat window answered by the AI (the website chat channel)
- Opening hours, location, booking requests

Orders from the website go into the same order flow as every other channel.

---

## 8. AI layer

The main part of the product. One engine, our tuned model, two faces.

### 8.1 Customer agent

- Talks to customers on every channel in Egyptian Arabic, Franco and English
- Answers menu, price, availability, hours and location questions from tools
- Takes orders and bookings, confirms them, tells the customer when the order is ready
- Recognizes regulars ("your usual?") and applies loyalty
- Hands off to staff when unsure, on complaints, or on anything outside its rules

### 8.2 Owner partner

- Answers the owner and staff: "how much did I sell today?", "what is my cost on
  a latte now?", "who owes me money?"
- Acts by talking: add an item, change a price, log waste, record a delivery
- Answers with the app's own cards and buttons, limited by the user's role

### 8.3 Proactive jobs (the AI manages, not only answers)

| Job | When | Output |
|-----|------|--------|
| Morning briefing | Every morning | What needs attention today |
| Stock running out | On `StockLow`, and by forecast | Draft purchase order for approval |
| Cash gap | On `CashGapFound` | Alert with the shift and the gap |
| Cost changed | On `CostChanged` | New cost per item and a price suggestion |
| Expiring items | On `StockExpiring` | Suggested discount |
| Regular went quiet | On `CustomerInactive` | Draft win-back message (term 2 sends) |
| Monthly value report | Every month | Money saved and earned, in pounds |

### 8.4 How much the AI does alone

The AI runs the daily work itself. The owner sets the rules once; only risky
actions wait for approval.

| Alone | Needs approval |
|-------|----------------|
| Answer customers, take normal orders, confirm, notify | Discounts above a limit, refunds |
| Log waste and sales the staff tell it about | Purchase orders above a limit |
| Draft purchase orders and price suggestions | Changing prices |
| Remind about debts (polite, rules from the owner) | Messages to many customers at once |

The limits live in the template and the business settings. Every AI action is
written to the audit log with `actor_type = ai`.

### 8.5 Business brain

Every AI answer reads three things:

1. **Profile:** what the business sells, its tone, its hours, its rules
2. **Tools:** live data from the modules (prices and stock always come from here)
3. **Memory:** what the owner decided before ("no delivery after 11", "stop
   selling the blueberry muffin")

### 8.6 Tools (starting set)

```
search_menu(query)                get_item(variant_id)
check_stock(stock_ref)            get_item_cost(variant_id)
create_order(customer, lines, type, channel)    get_order_360(number)
update_order_status(number, status)             list_orders(filter)
record_sale(lines, payment)       log_waste(stock_ref, qty, reason)
find_customer(name_or_phone)      get_customer_360(customer_id)
add_loyalty(customer_id, points)  create_booking(customer, time, size)
get_sales(period, group_by)       get_cash_status(shift)
draft_purchase_order(supplier, lines)           suggest_price(variant_id)
add_note(customer_id, text)       handoff_to_staff(conversation_id, reason)
send_message(conversation_id, text)
```

Each tool states what it needs, what it returns, which roles may use it, and
whether it needs approval. The tool list a business sees follows its capabilities
and the user's role.

### 8.7 The model

- **Our own model:** an open base model, fine-tuned by the team. No ready-made
  commercial model anywhere
- **One model for any category.** It learns to manage a small business through
  tools; the category comes from the template and the business brain, not from
  retraining
- **Showcase:** trained and tested deeply on coffee shops and small markets
- **Adaptation test:** measured on a category it saw little or nothing of in training
- **Method:** parameter-efficient fine-tuning (LoRA) of a small open model, on
  rented GPUs
- **Also open, also tunable:** speech-to-text for voice notes, an image model for
  menu photos and supplier invoices (term 2 for both if time is short)
- **Candidates to compare** (check for newer versions first): a strong tool-using
  family (for example Qwen3 4B to 8B) and an Egyptian-dialect model (for example
  Nile-Chat). Comparing both is part of the research

### 8.8 Training data

| Source | What |
|--------|------|
| Open Arabic tool-use datasets | Arabic and Egyptian requests mapped to tool calls |
| Open general tool-use datasets | The skill of choosing and filling tools |
| Our hand-written examples | Coffee shop and small market conversations, in the format we agree on |
| Shop visits and pilots | Real requests and messages, cleaned and anonymized |

One training example = the request, the tool calls with their inputs, the tool
results, and the final answer. Personal data is removed before training.

### 8.9 Guardrails

1. Exact data (prices, stock, totals) only comes from tools, never from memory
2. Tools follow capabilities and roles
3. Risky actions wait for approval
4. It asks when unsure ("which Mariam?")
5. It hands off on complaints and anything outside its rules
6. Personal data is masked in logs and training data
7. Measured on a test set, rerun after every change

### 8.10 Serving

The tuned model runs on our own GPU server (rented). The AI layer calls it, gets
tool calls back, runs them through the modules' tools, and returns the answer.

---

## 9. Studio, marketing and founder tools

Designed now, built mostly in term 2.

- **Studio:** brand kit (colors, fonts, logo composed from templates), menu and
  catalog design, store website look, social posts drafted by the AI
- **Marketing:** offers for slow hours, campaigns to segments, follow-ups and
  win-back messages, with consent checked before every send
- **Founder tools:** for new businesses: idea, validation, pricing, a simple
  projection, a launch checklist
- **GEO:** structured data and clean descriptions on the store website so AI
  search tools understand and recommend the business

---

## 10. Reports

- Event listeners keep summary tables (sales by hour, item, staff, channel)
- The numbers on Home come from the template
- Feeds the dashboard, the morning briefing and the AI's `get_sales`
- Read only

---

## 11. Frontend

| Area | Screens |
|------|---------|
| Home | To-do list from the AI, key numbers |
| Sales | Counter sales screen (fast, big buttons, works on a tablet or phone), open orders, shift open and close |
| Menu and stock | Items, recipes, ingredients, stock, waste, suppliers, purchases |
| Customers | List, customer page with history and loyalty, bookings |
| Inbox | All conversations, who is handling them (AI or staff), take over |
| AI partner | A chat panel on every page; answers as the app's own cards |
| Approvals | What the AI is waiting for the owner to approve |
| Settings | Business, members and roles, channels, AI rules, store website |
| Store website | Menu, ordering, chat window |

- Arabic right-to-left and mobile from day one
- The AI answers with the same card components the pages use
- The sales screen must stay fast at rush hour

---

## 12. Security

- Login with strong passwords, sessions, and lock-out after repeated failures
- Roles from data, not code; every screen and every AI tool checks them
- Row-level security on every business table, plus tests that try to cross businesses
- Audit log on every change, by people and by the AI
- AI safety: tests against prompt injection from customer messages; the AI can
  never do what the user or customer is not allowed to do
- Public website and chat: limits against spam and fake orders
- No secrets in the repository

---

## 13. Stack

Not decided. Each subteam picks its tools and can change them as long as the
contracts between parts hold. Things that affect the choice:

- The AI side will train and serve an open model, which usually means Python
  for training and a model server for running it
- Row-level security and transactions point to PostgreSQL, but this is open
- The sales screen must work on tablets and phones

---

## 14. What comes in which term

Nothing is dropped. Everything is ordered.

| Term 1 (first presentation) | Term 2 (full product) |
|-----------------------------|-----------------------|
| Core: businesses, members, roles, audit, templates (coffee shop, small market) | More templates and custom fields |
| ERP: menu, recipes, stock, waste, sales screen, orders, shifts and cash, suppliers and purchase drafts, cost per item | Offline sales, credit accounts in full, expiry tracking in full, multiple branches, e-invoicing |
| CRM: customers, identities, timeline, usual order, loyalty, notes | Segments, consent, merging, bookings in full |
| Channels: website chat and Telegram; WhatsApp if approved | WhatsApp in full, Instagram, voice notes, images |
| Store website: menu, ordering, chat | Studio design for the website, GEO |
| AI: customer agent, owner partner, morning briefing, alerts, approvals | All proactive jobs, monthly value report, win-back sends |
| Our tuned model powering the AI, test set, adaptation test | Speech and image models, larger training data, more categories |
| Home dashboard | Studio, marketing, founder tools, mobile app for notifications and quick management, API for existing websites, payment and courier integrations |

---

## 15. Research questions

1. Can one open model, fine-tuned on Egyptian Arabic small-business work, manage
   a business's customer chats and daily operations through tools, at a level
   owners accept?
2. Does the same model adapt to a new business category with configuration and
   little extra data?

**Measured:**

| Measure | How |
|---------|-----|
| Chats handled with no human | Share of conversations in the pilot |
| Right action | Correct tool and inputs on the test set |
| Orders captured from chat | Share of chat orders created correctly |
| Reply time | Seconds |
| Owner time and money saved | Pilot, compared with a baseline week before the AI is on |
| Tuned versus untuned | Same test set, our tuned model against the untuned base model |
| Adaptation | Score on a category barely seen in training |

---

## 16. Risks

| Risk | What we do |
|------|-----------|
| Our tuned model is not reliable enough in time | Start data and tuning in week 4; narrow tools; hand off when unsure; measure weekly |
| Meta approval for WhatsApp is slow | Website chat and Telegram first; request approval in week 4 |
| Wrong price or order from the AI | Prices and stock only from tools; confirmation with the customer; hand off below a confidence level |
| Scope creep | Work goes by term; one person has the final say on scope |
| Data leaks between businesses | Row-level security and tests that try to cross businesses |
| Shops do not adopt it | Built from shop visits; fast sales screen; visible monthly value |
| GPU cost | Small models, rented by the hour, budget planned in week 5 |
