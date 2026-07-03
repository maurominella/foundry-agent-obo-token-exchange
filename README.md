# Propagating the Teams User Identity to a Microsoft Foundry Agent

**On-Behalf-Of (OBO) and Token Exchange for Foundry agents on Agent 365**

> Flow the signed-in Microsoft Teams user identity through to a Foundry hosted agent (Agent 365) and on to an Entra-protected downstream service (e.g. Microsoft Graph), via OBO / token exchange — with no consent or login prompts.

**Repository:** `foundry-agent-obo-token-exchange`

```bash
git clone https://github.com/maurominella/foundry-agent-obo-token-exchange.git
```

> **Author:** Mauro Minella — Senior Cloud Solution Architect, Microsoft
> **Status:** Validated end-to-end in a development environment
> **Scope:** How to carry the identity of the user signed in to Microsoft Teams all the way from the Teams client, through a custom bot and a Microsoft Foundry hosted agent (published to the Agent 365 registry), down to an Entra-protected backend that the agent calls on the user's behalf.

---

## Introduction

When a user signs in to Microsoft Teams, their identity is already authenticated through Entra ID. However, when the user starts a conversation with an agent published on Microsoft Agent 365, that identity has to be made available along the **whole application chain**: from the Teams client, to the Bot Service, to the agent, and finally to the downstream services the agent uses to answer.

The identity has to be carried across **three hops**:

1. From **Teams** to the **bot**
2. From the **bot** to the **agent**
3. From the **agent** to the **downstream services**

Identity travels inside tokens made of claims — among them the **audience** (`aud`), which identifies the intended recipient of the token. Each hop therefore needs a token with a **different audience**: one for Teams, one for the bot, one for the agent, and one for each downstream service. This chain of conversions — called **token exchange** — is what lets each component operate **On Behalf Of the user (OBO)**.

This documentation analyses how that flow applies to **Microsoft Foundry agents published on Microsoft Agent 365**, highlighting the structural limits of the Activity Protocol and the implications for token exchange — and shows the working, validated end-to-end solution.

### How to read this

- If you only want the **outcome**, read the [Final Result](docs/00-final-result.md).
- If you want the **concept**, read chapters [01](docs/01-overview-and-challenge.md), [02](docs/02-authentication-flow-and-prerequisites.md), [05](docs/05-two-identity-axes.md) and [06](docs/06-five-token-model.md).
- If you are **implementing**, read everything in order; chapters 03–10 are the build steps.

> **Note on values.** The identifiers, GUIDs, endpoints, and dev-tunnel URLs shown throughout are taken verbatim from the validated development environment. **Secrets are masked.** Substitute your own tenant, application, and project values when reproducing the setup.

---

## Table of Contents

| # | Chapter | What it covers |
|---|---------|----------------|
| ⭐ | [Final Result](docs/00-final-result.md) | The outcome: the full silent Teams → agent → Graph round-trip |
| 01 | [Overview & the Challenge](docs/01-overview-and-challenge.md) | Bot Service, the Activity Protocol limit, the custom bot, the dev tunnel |
| 02 | [Authentication Flow & Prerequisites](docs/02-authentication-flow-and-prerequisites.md) | Prompt vs hosted agent, the three app registrations, Entra claims, the token list |
| 03 | [Token A — Teams SSO](docs/03-token-a-teams-sso.md) | The incoming token: the user calls the bot, not the agent |
| 04 | [Token B — Foundry Access](docs/04-token-b-foundry-access.md) | Two approaches (user-delegated vs app-only) and the bot code |
| 05 | [Two Identity Axes](docs/05-two-identity-axes.md) | Why two registrations: conversational identity vs Foundry access |
| 06 | [The Five-Token Model](docs/06-five-token-model.md) | The canonical names for every token |
| 07 | [Token C — The User Assertion](docs/07-token-c-user-assertion.md) | The critical part: scope, identity, tool + the agent OBO code |
| 08 | [Token D — The Downstream OBO](docs/08-token-d-downstream-obo.md) | The agent mints the downstream token on behalf of the user |
| 09 | [Publishing the App to Teams](docs/09-publishing-to-teams.md) | Manifest, package ZIP, sideload, verify |
| 10 | [Wiring Token C & Verification](docs/10-wiring-and-verification.md) | Forwarding Token C in code + the expected token claims |
| A | [Appendix — The Three Registered Applications](docs/11-appendix-app-registrations.md) | Per-app reference: Overview, Certificates & secrets, Expose an API, API Permissions |

---

## Repository layout

```
foundry-agent-obo-token-exchange/
├── README.md                     ← you are here (introduction + navigation)
├── CHANGELOG.md
└── docs/
    ├── 00-final-result.md         ← the outcome we achieve
    ├── 01-overview-and-challenge.md
    ├── 02-authentication-flow-and-prerequisites.md
    ├── 03-token-a-teams-sso.md
    ├── 04-token-b-foundry-access.md
    ├── 05-two-identity-axes.md
    ├── 06-five-token-model.md
    ├── 07-token-c-user-assertion.md
    ├── 08-token-d-downstream-obo.md
    ├── 09-publishing-to-teams.md
    ├── 10-wiring-and-verification.md
    ├── 11-appendix-app-registrations.md   ← Appendix A index (the 3 applications)
    ├── 11a-app-bot.md                      ← Appendix A.1 (Bot)
    ├── 11b-app-foundry-access.md           ← Appendix A.2 (Foundry Access)
    ├── 11c-app-downstream-obo.md           ← Appendix A.3 (Downstream / App-OBO)
    └── images/                    ← all screenshots referenced by the chapters
```

---

## Maintaining this documentation

**Source of truth.** This repository is the *canonical, published* version of the documentation. Personal working notes (kept separately, in the author's OneNote) are the raw source that feeds into it — they are **not** part of this repo and are never maintained as a parallel deliverable. The flow is one-directional: notes → repository.

**Nothing changes automatically.** There is **no live sync** between the notes and this repository. The repo changes *only* when a commit is deliberately made and pushed, after reviewing the diff. Editing the private notes never alters what readers of this repository see.

**Stability contract (so readers and clients are never surprised).** This documentation is shared with external readers, so its structure is treated as a stable, published interface:

- **Filenames are stable.** The `docs/NN-*.md` names do not change once published — renaming a file would break external links and bookmarks.
- **Chapter numbers, titles, and order are stable.** New material is added *within* the existing structure (a new subsection in the relevant chapter, or a new appendix) — never by renumbering or reshuffling what already exists.
- **Changes are additive by default.** Corrections and additions are preferred over rewrites. A structural change (rename, reorder, split, merge) happens **only when explicitly intended**, and is recorded in `CHANGELOG.md`.
- **Every change is reviewed before push.** The Git diff is inspected so each update is intentional and narrowly scoped — never a full regeneration of the repository.

**How to update, in practice.**
1. **Small fix** (a value, a GUID, a typo) → edit the specific `.md` directly (even from the GitHub web editor). Commit.
2. **New content** → draft it in the private notes, then consolidate it into the *relevant existing chapter* as a focused diff. Commit.
3. **Record it** → note anything reader-visible in `CHANGELOG.md`, and tag releases (`v1.0`, `v1.1`, …) so readers see intentional, dated updates rather than silent churn.
