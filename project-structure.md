# Project structure

How the code is laid out: the repos and their folders. **What** the system is
(the idea, the tools, the skills, the AI, the modules, the windows, the flows)
is in [technical-structure.md](technical-structure.md). This file is section 21
of it, kept on its own page so each subteam can find its folders quickly; when
one changes, both change. Numbers in parentheses, like (4.4), are sections of
`technical-structure.md`.

Open questions are in section 26 of `technical-structure.md`.

## 1. The repos

| Repo | Holds | Who |
|------|-------|-----|
| `app` | Contracts, backend and web, in one repo | Backend, Frontend |
| `app-ai-agent` | The AI layer: faces, jobs, planner, skills | AI agent |
| `app-ai-model` | Training, test set, results, serving | AI data |

Contracts, backend and web share one repo: a shared value changes in one commit,
with no release step. The two AI repos are separate because they have their own
tools, GPUs and pace; they read the contracts from `app`.

## 2. `app`

```
app/
├── README                              what it is, how to run everything
├── rules/                              one page each: modules, data, tools, events, web, contracts
├── contracts/                          written once, used by backend, web and the AI repos
│   ├── tools/                          one folder per module, one reference page per tool (4.4)
│   │   ├── core/  erp/  crm/  channels/  store/
│   │   ├── studio/  marketing/  team/  reports/  ai/
│   │   └── (each page: purpose, inputs, output, errors, effects, rules, examples, version)
│   ├── packs/                          one file per pack (7.3): module, needs, tools, windows,
│   │                                   skills, listeners, settings, setup skill
│   ├── cards/                          order, item, stock, customer, cash, sales, approval, plan, post
│   ├── events/                         one file per event: the envelope (2.6) and its data
│   ├── values/                         ID types, order statuses, roles and permissions, channels,
│   │                                   payment methods, AI permission settings
│   ├── errors/                         error codes
│   ├── limits/                         password, code expiry, lock-out, upload size
│   ├── training/                       training example format: request, skill, tool calls with
│   │                                   inputs, tool results, final answer (shared by agent and model)
│   └── changes/                        every change to a tool, event or card, announced before merging
├── backend/
│   ├── config/                         settings read once at start; example settings file
│   ├── templates/                      coffee-shop, small-market: packs on, fields, stages,
│   │                                   home items, AI defaults, template skills
│   ├── database/
│   │   ├── migrations/                 one set per schema: core, erp, crm, channels, store,
│   │   │                               studio, marketing, team, reports, ai
│   │   └── seeds/                      invented demo coffee shop and small market
│   ├── platform/
│   │   ├── tenancy                     every call inside one business (row-level security)
│   │   ├── auth                        sign up, log in, sessions, lock-out
│   │   ├── permissions                 one check for every tool, for the app and the AI alike
│   │   ├── packs                       essentials and packs on for this business; needs, on, off
│   │   ├── tool-registry               loads the tool pages; filters by pack, role, face
│   │   ├── context                     business, actor, correlation ID, retry key on every call (2.6)
│   │   ├── audit                       audit row on every write
│   │   ├── events                      outbox, dispatcher, retries, listener registry
│   │   ├── scheduler                   time-based events: expiring stock, quiet regulars,
│   │   │                               morning briefing, scheduled posts
│   │   ├── idempotency                 retried calls write once
│   │   ├── undo                        runs a tool's undo from the audit log
│   │   ├── dry-run                     runs tools with no real writes, for skill test runs
│   │   ├── numbering                   #1042 per business
│   │   ├── money                       whole piastres plus currency
│   │   ├── errors                      error codes, never message text
│   │   ├── streaming                   relays the AI's streamed answer to the app
│   │   └── realtime                    live updates: orders board, inbox, stock
│   ├── api/                            one tool API, four doors (section 3)
│   │   ├── app/                        logged-in members (session); relays owner chat to the AI
│   │   ├── ai/                         the AI layer (service token plus who it acts for)
│   │   ├── public/                     shop pages, chat widget, channel webhooks; rate limits
│   │   └── outside/                    outside websites with API keys (term 2)
│   ├── modules/
│   │   ├── core/                       businesses, members, roles, locations, templates, packs,
│   │   │                               notifications
│   │   ├── erp/
│   │   │   ├── catalog/                items, variants, modifiers, categories (essentials)
│   │   │   ├── orders/                 orders, lines with snapshots, the order stages
│   │   │   ├── payments/               payments, refunds
│   │   │   ├── sales-screen/           counter sales
│   │   │   ├── shifts/                 shifts, cash movements, cash gap
│   │   │   ├── stock/                  movement ledger, levels cache, waste, forecast, rebuild
│   │   │   ├── recipes/                ingredients, recipe lines, cost per item
│   │   │   ├── purchasing/             suppliers, purchase orders, receiving
│   │   │   ├── credit/                 customer debts
│   │   │   ├── expenses/
│   │   │   └── adapters/               payments, couriers (term 2)
│   │   ├── crm/
│   │   │   ├── customers/              customers, stats cache, usual order, merge and forget
│   │   │   ├── identities/             resolve identity, phone to +20, merge suggestions
│   │   │   ├── timeline/               activity from other modules' events
│   │   │   ├── loyalty/
│   │   │   ├── support/                cases, case events, vouchers, follow-ups
│   │   │   ├── bookings/
│   │   │   ├── engagement/             notes, tags
│   │   │   └── segments/               segments and consent (term 2)
│   │   ├── channels/
│   │   │   ├── inbox/                  incoming queue, duplicates dropped
│   │   │   ├── conversations/          AI or staff, take over, hand back, tags
│   │   │   ├── messages/               one message shape
│   │   │   ├── auto-replies/           away, opening hours, greeting
│   │   │   └── adapters/               website chat, Telegram; WhatsApp, Instagram, Facebook later
│   │   ├── store/
│   │   │   ├── site/                   shop page, pages, banner, menu display, public ordering
│   │   │   ├── search-presence/        descriptions, structured data, score (term 2)
│   │   │   └── adapters/               Google business profile and reviews (term 2)
│   │   ├── studio/                     brand kit, design templates, render, designs, posts
│   │   │   └── adapters/               image model (term 2)
│   │   ├── marketing/                  offers, campaigns, sends (term 2)
│   │   ├── team/                       staff profiles, rota, attendance, tasks (term 2)
│   │   └── reports/                    summary tables from events
│   │   (each module folder: tools as its only public face, service, data, listeners, tests)
│   ├── fake/                           answers every tool from its page's examples and the
│   │                                   demo businesses, for the frontend and the AI team (6.10)
│   └── tests/
│       ├── isolation                   business A reads business B: must fail
│       ├── schema-users                each module's database user sees only its schema
│       ├── roles                       each role against each tool
│       ├── tools                       each tool: allowed, refused by role, by pack, by face
│       ├── pages                       each tool behaves exactly as its reference page says
│       ├── packs                       essentials alone, each template, each pack with its needs
│       ├── undo                        every write tool's undo puts things back
│       ├── order-flow                  every allowed and refused order move
│       ├── processes                   each cross-module process (2.7) end to end
│       └── rebuild                     stock levels and customer stats rebuilt from history
└── web/
    ├── shared/
    │   ├── cards/                      one component per card: used in pages and in AI answers
    │   ├── chat/                       message list, streaming, input box, + menu, threads,
    │   │                               plan and approval cards
    │   ├── layout/                     sidebar (by role and packs), side panel for opened cards
    │   ├── tools-client/               calls the same tools the AI uses
    │   ├── i18n/                       Arabic and English (Arabic first, right to left)
    │   └── tokens/                     colours, spacing, fonts
    ├── staff/                          windows 1 to 12 (section 17.2)
    │   ├── setup/  home/  sales/  orders/  menu-stock/  customers/
    │   ├── inbox/  approvals/  settings/                       (term 1)
    │   └── team/  marketing/  website-search/                  (term 2)
    ├── shop/                           windows 13 and 14: shop page with chat, order status
    └── landing/                        window 15
```

## 3. `app-ai-agent`

Folders only. The AI team fills them.

```
app-ai-agent/
├── faces/                              customer agent, owner partner
├── jobs/                               scheduled and event-driven work
├── planner/                            plans, approval per plan, running, checking, undo
├── skills/                             core skills
├── tools/                              tool loading, find_tools
├── model/                              talking to the models
├── policy/                             permission settings (6.6), hand-off triggers, injection checks
├── brain/                              profile, knowledge, memory, threads
├── approvals/
├── log/                                AI action log, masking
└── tests/
    ├── fakes/                          fake model and fake tools built from the tool pages' examples
    ├── conversations/                  scripted conversations with the expected tool calls
    └── injection/                      prompt-injection cases
```

## 4. `app-ai-model`

Folders only. The AI team fills them.

```
app-ai-model/
├── configs/  testset/  devset/  data/
├── prepare/  train/  evaluate/  serve/
└── runs/  results/
```

