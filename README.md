# Graduation Project

**One AI for every small business.** Each business gets its own AI manager,
running on our own tuned open models, that talks to customers, takes orders,
watches stock and cash, and tells the owner what needs attention. It does all of
it through the tools of an open ERP and CRM. Showcase: coffee shops and small
markets in Egypt, in Egyptian Arabic, Franco and English.

This repository holds the project's documents: the design, the code layout and
the term 1 plan. The code itself lives in three other repos ([below](#the-code)).

## Documents

| Document | What it is | Status |
|----------|-----------|--------|
| [Technical structure](technical-structure.md) | The design of the whole product, both terms: what it is, who it is for, how it is built, what each part owns, how the AI runs the business | **The single source of truth** |
| [Project structure](project-structure.md) | The repos and their folders. Section 21 of the technical structure, on its own page so each subteam finds its folders quickly | Mirrors section 21 |
| [Term 1 plan](plan-first-semester.md) | What each subteam delivers in term 1, and roughly when, and the final output for the first presentation | A reference, not a list of deadlines |
| [Archive](archive/) | Old decisions that were discarded, kept for the record | **Not current. Do not use** |

### Where to start

- **New to the project:** sections 0 and 1 of the
  [technical structure](technical-structure.md), then section 1 of the
  [term 1 plan](plan-first-semester.md)
- **Building something:** your subteam's row in section 3 of the plan, your
  folders in the [project structure](project-structure.md), then the design
  sections those folders point to
- **Looking for a decision that is still open:** section 26 of the technical
  structure

## The idea in four lines

From section 0 of the technical structure:

1. **The backend is an open ERP and CRM whose only door is its tools.** The app,
   the AI and later outside websites use the same tools, with the same
   permissions and the same audit
2. **Tools say what is possible. Skills say how to do it well.** Tools are few
   and stable; skills are written know-how that the owner can change by talking
3. **Use only what you need.** Every business gets the essentials; the rest is a
   library of packs it turns on and off. A template is a recommended set of packs
4. **Our own tuned models.** Open models fine-tuned by the team. No ready-made
   commercial model anywhere. The owner decides what the AI may do alone

## The code

| Repo | Holds | Who |
|------|-------|-----|
| [`app-backend`](https://github.com/puagraduationproject/app-backend) | Contracts, platform, modules, database, fake backend, tests | Backend |
| [`app-web`](https://github.com/puagraduationproject/app-web) | Staff app, shop pages, landing page | Frontend |
| `app-ai-agent` | The AI layer: faces, jobs, planner, skills | AI agent |
| `app-ai-model` | Training, test set, results, serving | AI data |

The folders inside each are in the [project structure](project-structure.md).

## Team

10 members in 6 subteams: Backend, Frontend, AI agent, AI data, Market and
Security. Who owns what is in section 3 of the [term 1 plan](plan-first-semester.md).

## Timeline

| Term | Weeks | Milestone |
|------|-------|-----------|
| Term 1 | Weeks 4 to 14 (October to December 2026) | Practice presentation in week 10; first presentation after week 14 |
| Term 2 | | The full product (section 23 of the technical structure) |

## Changing these documents

1. A change to what the system is goes into the
   [technical structure](technical-structure.md) first
2. If it moves repos or folders, section 21 and the
   [project structure](project-structure.md) change in the same commit
3. The [term 1 plan](plan-first-semester.md) follows the design. When they
   disagree, the design wins and the plan is updated
4. Changes reach `main` through a pull request with one review
5. A retired document moves into `archive/` whole. Nothing is taken back out of it

## Never in any repo

No server information goes into any of the team's repos, in files, commit
messages, issues or pull requests:

- IP addresses, server names, hosting providers or server plans
- Test or deploy addresses and domains
- Server users, folders, ports, limits or how a server is set up
- Keys, tokens, passwords or settings files with real values

The design says only "our server" and "the test address". The real details are
shared with the team outside GitHub. If one is pushed by mistake, it is removed
from the history, not just from the latest version.