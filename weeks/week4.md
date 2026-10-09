# Week 4 (Oct 10 to 16): Agreements and foundations

> **Goal of the week:** every subteam knows exactly what it builds, and all the
> parts agree with each other on paper before anyone builds them.
>
> **End of the week:** all leads sign off the first `contracts.md`.
>
> Full plan: [plan-term1.md](../plan-term1.md)

---

## Everyone

- [ ] Join the GitHub organization and accept your team invite
- [ ] Read the [README](../README.md) and the [term 1 plan](../plan-term1.md)
- [ ] Each subteam picks its **lead** and tells the group
- [ ] Agree on the weekly **leads meeting** time (30 minutes)
- [ ] Agree on the weekly **integration day** (last working day of the week)
- [ ] Agree on one group chat for questions between subteams

---

## Backend (2 CS)

- [ ] **What the system stores:** a list of every kind of record with its main
      fields. At least: business, member, role, product, product variant (size,
      color), category, stock movement, customer, order, order line, payment,
      expense. Written in `contracts.md`
- [ ] **Order statuses:** draft, confirmed, packed, shipped, delivered,
      cancelled, returned. Which moves are allowed, and what happens on each
      (for example: confirmed reserves stock, cancelled gives it back)
- [ ] **Money rules:** amounts stored as whole numbers in the smallest unit
      (piastres), always with a currency. Discounts, totals and refunds written
      as simple rules
- [ ] **In-store sale:** what is different from an online order (no shipping,
      paid on the spot, stock goes down right away)
- [ ] **Shared place to work:** the project skeleton in the repo, with a short
      guide so any member can run it on their own machine
- [ ] Write down which tools Backend chose, and why (can change later)

**Hands to others:** the list of stored records (to Frontend, AI agent, Security).

---

## Frontend (2 CS)

- [ ] **Wireframes** (simple sketches are fine) for every term 1 screen:
  - [ ] Sign up and log in
  - [ ] App frame: menu, language switch (Arabic and English), AI panel
  - [ ] Dashboard
  - [ ] Products: list, add, edit. Stock view
  - [ ] Orders: list, order page with status bar. Quick-sale screen
  - [ ] Customers: list, customer page with history
  - [ ] Store settings for the owner
  - [ ] Store website: home, product page, cart, checkout (cash on delivery)
- [ ] Every screen works **right to left** in Arabic; key screens have a
      **mobile** sketch
- [ ] **Shared cards** list: order card, customer card, product card. These are
      the cards the AI will reuse in its answers
- [ ] **Landing page** design and text, in Arabic and English, with the
      sign-up form from Market
- [ ] Write down which tools Frontend chose, and why (can change later)

**Hands to others:** wireframes and the shared cards list (to everyone).

---

## AI agent (2 AI)

- [ ] **AI tool list** (10 to 15 tools). For each tool: name, what it does,
      what it needs, what it returns, which roles may use it, and whether it
      needs the owner's confirmation. Starting list:
  - Search products, get product details, check stock, add a product,
    change a price
  - Find a customer, get customer history, add a note, add a task
  - Create an order, get order details, list orders, change order status,
    record an in-store sale
  - Get sales numbers (by day, week, month, product)
- [ ] **Confirmation rule:** which actions always need the owner to confirm
      (anything that saves or changes data) and which do not (questions)
- [ ] **3 example conversations** from start to end (at least one in Arabic),
      showing the tools used and the card shown at the end

**Hands to others:** the AI tool list (to Backend, Security, AI data).

---

## AI data (2 AI)

- [ ] **Shortlist of 3 ready-made models** to compare against, with why each
      one: Arabic quality, how well it uses tools, speed, cost
- [ ] **Shortlist of 2 to 3 open base models** we could fine-tune (small
      enough to train on a rented GPU, good with Arabic and with tools)
- [ ] **Training data format:** how one training example looks (the owner's
      request, the tool the AI should call with its inputs, and the final
      answer), so everyone who collects data uses the same shape
- [ ] **30 example requests** (15 Arabic, 15 English), the way a real shop
      owner would type them. For each: the right tool and the right inputs.
      Example: *"عايز أعرف فاضل كام هودي أسود L"* means check stock, black
      hoodie, size L
- [ ] **How we score the AI:** what counts as correct (right tool, right
      inputs, right final answer) and what counts as a failure

**Hands to others:** the example requests (to AI agent).

---

## Security (1)

- [ ] **Roles and permissions** table. Suggested roles: owner, manager, sales,
      viewer. For each role, yes or no for: see profit, give discounts, delete
      products, change prices, invite members, cancel orders, use each AI action
- [ ] **Main risks** list, each with how we will stop it:
  - One business sees another business's data
  - A stolen password or session
  - Staff misusing discounts or cancelling orders
  - Spam or fake orders on the store website
  - The AI tricked into doing something the user is not allowed to do
  - Passwords or keys pushed to GitHub by mistake
- [ ] **GitHub rules** to propose to the group: no secrets in the repo,
      changes to `main` only through a pull request with one review

**Hands to others:** the roles table (to everyone).

---

## Market (1 AI lead, everyone helps)

- [ ] **Interview script** (about 15 minutes): how they track stock, orders
      and customers today; what tools they use; what hurts most; would they
      try a new system; what they would pay
- [ ] **List of 50 target shops:** physical shops and online stores. For each:
      name, type, area, contact, and which member will visit
- [ ] **Sign-up form** for the waiting list: name, shop name, shop type,
      phone, city, what they want most
- [ ] Split the shops so each member has at least 3 visits for weeks 5 to 8

**Hands to others:** the sign-up form (to Frontend for the landing page).

---

## Integration day: sign off `contracts.md`

All leads meet and check that everything matches:

- [ ] Every screen in the wireframes shows data that Backend stores
- [ ] Every AI tool uses data and actions that Backend provides
- [ ] Every action on a screen or in the AI has a role rule from Security
- [ ] Every example request from AI data maps to a tool in the tool list
- [ ] Open questions written down, each with an owner

When all boxes are ticked, the leads sign off. Changes after this need every
subteam they touch to agree.

---

## Done this week when

- [ ] `contracts.md` exists and is signed off by all leads
- [ ] Wireframes, tool list, roles table and example requests are in the repo
- [ ] Interview script, shop list and sign-up form are ready for week 5
- [ ] Every member can run the project skeleton on their machine
