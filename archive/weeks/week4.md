# Week 4 (Oct 10 to 16): Agreements and foundations

> **Goal of the week:** every subteam knows exactly what it builds, and all the
> parts agree with each other on paper before anyone builds them.
>
> **End of the week:** all leads sign off the first `contracts.md`.
>
> Full plan: [plan-first-semester.md](../plan-first-semester.md) · Design: [technical-structure.md](../technical-structure.md)

---

## Everyone

- [x] Join the GitHub organization and accept your team invite
- [x] Read the [technical structure](../technical-structure.md) sections 0 and 1,
      and the [term 1 plan](../plan-first-semester.md)
- [ ] Each subteam picks its **lead**; the group picks **one person with the
      final say on scope**
- [ ] Agree on the weekly **leads meeting** time (30 minutes) and the weekly
      **integration day** (last working day)
- [ ] Agree on one group chat for questions between subteams
- [ ] Each member visits **one coffee shop this week** and notes what wastes the
      owner's time and money (feeds Market and AI data)

---

## Backend (2 CS)

- [X] **Account** sing-up & login flow - DB 
- [ ] **What the system stores:** every kind of record with its main fields, in
      `contracts.md`. At least: business, member, role, template, item, variant,
      modifier (milk type, extra shot), ingredient, recipe line, stock movement,
      order, order line, payment, shift, supplier, purchase order, customer,
      customer identity, loyalty, conversation, message
- [ ] **Order stages** for the coffee shop: new, accepted, preparing, ready,
      collected or delivered, cancelled. Which moves are allowed and what happens
      on each (accepted reserves stock, completed deducts recipe ingredients)
- [ ] **Counter sale:** created and completed in one step, deducts stock right away
- [ ] **Shift rules:** open with cash, close with counted cash, gap = counted
      minus expected
- [ ] **Money rules:** whole piastres plus currency; discounts, totals, cost per
      item from recipes
- [ ] **Events list:** which events each part emits (`OrderCreated`, `StockLow`,
      `CashGapFound`, `CostChanged`, `MessageReceived`, ...) and who listens
- [ ] **Shared place to work:** project skeleton in the repo with a short guide
      so any member can run it
- [ ] Write down which tools Backend chose, and why (can change later)

**Hands to others:** records, events and order stages (to Frontend, AI agent, Security).

---

## Frontend (2 CS)

- [ ] **Wireframes** (simple sketches are fine) for every term 1 screen:
  - [ ] Sign up, log in, choose business type
  - [ ] App frame: menu, language switch (Arabic and English), AI panel
  - [ ] **Counter sales screen** (big buttons, fast, tablet and phone)
  - [ ] Open orders board (pickup and delivery) and order page
  - [ ] Menu: items, sizes and options, recipe editor
  - [ ] Ingredients, stock, waste log, suppliers, purchase drafts
  - [ ] Shift open and close with cash count
  - [ ] Customers: list, customer page with history and loyalty
  - [ ] Inbox: conversations, who is handling them (AI or staff), take over
  - [ ] Approvals: what the AI is waiting for
  - [ ] Home: the AI's to-do list and key numbers
  - [ ] Settings: business, members and roles, channels, AI rules
  - [ ] Store website: menu, ordering, chat window
- [ ] Every screen works **right to left** in Arabic; key screens have a
      **mobile** sketch
- [ ] **Shared cards** list: order, item, stock, customer, cash. The AI reuses them
- [ ] **Landing page** design and text in Arabic and English, with the sign-up form
- [ ] Write down which tools Frontend chose, and why (can change later)

**Hands to others:** wireframes and the shared cards list (to everyone).

---

## AI agent (2 AI)

- [ ] **AI tool list** (start from section 8.6 of the technical structure). For
      each tool: name, what it does, what it needs, what it returns, which roles
      may use it, and whether it needs approval
- [ ] **What the AI does alone and what needs approval** (start from section 8.4):
      the limits for discounts, refunds, purchase orders and price changes
- [ ] **Handoff rules:** when the customer agent passes the chat to staff
      (complaints, unclear requests, anything outside the rules)
- [ ] **3 example conversations** from start to end, with the tools used:
  - [ ] A customer orders 2 lattes and a croissant for pickup on chat (Arabic)
  - [ ] The owner asks for today's sales and the cost of a latte (Arabic)
  - [ ] Stock runs low and the AI drafts a purchase order for approval
- [ ] **Morning briefing:** what it contains and in what order

**Hands to others:** the AI tool list (to Backend, Security, AI data).

---

## AI data (2 AI)

- [ ] **Shortlist of open base models** to tune (no ready-made commercial
      models): at least one strong at using tools and one strong in Egyptian
      Arabic. Check for the newest versions. For each: size, Arabic quality, tool
      use, license, GPU needed
- [ ] **Training example format:** the request, the tool calls with inputs, the
      tool results, the final answer. Everyone who collects data uses it
- [ ] **Open datasets** to start from: Arabic tool-use data and general tool-use
      data, with their licenses
- [ ] **30 example requests** from coffee shop customers and owners, in Egyptian
      Arabic, Franco and English, each with the right tool and inputs. Example:
      *"عايز ٢ لاتيه وكرواسون تيك أواي"* means create order, pickup, 2 lattes,
      1 croissant
- [ ] **How we score the model:** right tool, right inputs, right final answer,
      handoff when it should
- [ ] **GPU plan:** where to rent, which GPU for training and for running, rough
      cost per month

**Hands to others:** example requests and data format (to AI agent and Market).

---

## Security (1)

- [ ] **Roles and permissions** table: owner, manager, staff, viewer. For each,
      yes or no for: see profit and costs, give discounts, cancel orders, open and
      close shifts, change prices, approve purchase orders, invite members, and
      each AI action
- [ ] **Main risks** list, each with how we stop it:
  - One business sees another business's data
  - A stolen password or session
  - Staff misusing discounts, cancellations or the cash drawer
  - Spam or fake orders on the website and chats
  - A customer message tricking the AI (prompt injection)
  - Personal data leaking into training data or logs
  - Passwords or keys pushed to GitHub
- [ ] **GitHub rules** to propose: no secrets in the repo, changes to `main` only
      through a pull request with one review

**Hands to others:** the roles table (to everyone).

---

## Market (1 AI lead, everyone helps)

- [ ] **Interview script** (about 15 minutes) for coffee shops and small markets:
      how they track stock, cash, orders and customers today; how they take chat
      and delivery orders; what costs them the most money; delivery app fees;
      would they try a new system; what they would pay
- [ ] **List of 50 target shops** (coffee shops first, then small markets): name,
      type, area, contact, and which member visits
- [ ] **Sign-up form** for the waiting list: name, shop name, type, phone, city,
      biggest problem
- [ ] **Consent text** for using anonymized messages from shops to train our model
- [ ] **Start the WhatsApp business request** and create the Telegram bot
- [ ] Split the shops so each member has at least 3 visits for weeks 5 to 8

**Hands to others:** sign-up form (to Frontend), consent text (to AI data).

---

## Integration day: sign off `contracts.md`

All leads meet and check that everything matches:

- [ ] Every screen shows data that Backend stores
- [ ] Every AI tool uses data and actions that Backend provides
- [ ] Every action on a screen or by the AI has a role rule from Security
- [ ] Every example request maps to a tool in the tool list
- [ ] The events list covers every proactive AI job (low stock, cash gap, cost change)
- [ ] Open questions written down, each with an owner

When all boxes are ticked, the leads sign off. Changes after this need every
subteam they touch to agree.

---

## Done this week when

- [ ] `contracts.md` exists and is signed off by all leads
- [ ] Wireframes, tool list, roles table, data format and example requests are in the repo
- [ ] Base model shortlist and GPU plan ready for week 5
- [ ] Interview script, shop list, sign-up form and consent text ready for week 5
- [ ] WhatsApp request started, Telegram bot created
- [ ] Every member can run the project skeleton on their machine
