# ⭐ Final Result

> This page describes **the outcome** that the rest of this documentation builds toward. Read it first so that every configuration step in the following chapters has a clear purpose.

## What we achieve

A user opens **Microsoft Teams**, types a message to a bot, and that message travels — **through a custom bot, into a Microsoft Foundry hosted agent, and out to Microsoft Graph** — with the agent calling Graph **on behalf of that exact Teams user**.

And the defining property of the result:

> **Not a single consent screen. Not a single login prompt. Nothing.**
> The whole chain runs silently, starting from nothing more than the identity of the user already signed in to Teams.

This is **validated working end-to-end**: from Teams, to the bot, to the agent that invokes Microsoft Graph authenticating *on behalf of* the connected Teams user — without ever asking for a login or a consent.

## What happens underneath (the silent round-trip)

```text
User in Teams:   "read my recent files"
        │
        ▼   (no login, no consent, no popup — it just works)
Teams ── Token A ──► Bot ──► Foundry agent ──► Microsoft Graph  (as the signed-in Teams user)
                      │                │
                      ├─ Token B ──────┘  (app-only, Authorization header, validated then stripped)
                      └─ Token C ──────────────► forwarded to the agent → OBO → Token D
        │
        ▼
Agent reply:     "Here are your files, <User Name>: ..."
```

1. **Teams → Bot.** The signed-in Teams session silently yields **Token A** (Teams SSO, `aud = api://botid-<Bot>`, `scp = access_as_user`). No prompt — the Teams clients are pre-authorized on the bot's scope.
2. **Bot mints two tokens:**
   - **Token B** — an app-only token (`aud = https://ai.azure.com`) that passes Foundry's RBAC gate. It carries **no** user identity and is discarded by Foundry after the check.
   - **Token C** — a user-delegated token (`aud = api://app-obo/…`) obtained by exchanging Token A. It **is** the user's identity, and travels in the custom header **`x-ms-user-assertion`**. No prompt — the Bot is pre-authorized on the App-OBO scope.
3. **Bot → Foundry (OpenAI Responses protocol).** Token B goes in `Authorization`; Token C goes in the custom header. Foundry validates Token B, strips it, and passes the custom header through to the agent.
4. **Agent → Microsoft Graph.** The hosted agent (Python) reads Token C from `x-ms-user-assertion`, builds a confidential client application with the **App-OBO** credentials, and performs `acquire_token_on_behalf_of` to obtain **Token D** — an Entra token for Microsoft Graph, issued **for the original Teams user**. It then calls Graph as that user.

The complete delegated identity chain is:

```text
Pre-token  →  Token A  →  Token C  →  Token D
 (Teams)      (SSO)       (user)      (Graph, on behalf of the user)
```

Token B is a **side branch** — it only opens Foundry's door (RBAC) and is thrown away; it is never part of the identity chain.

## Why there is zero friction (the "no prompt" guarantee)

The silence is designed, not accidental. Two pre-authorizations remove every possible consent prompt, and both On-Behalf-Of exchanges run **server-side** (so they *cannot* show a popup — the required consents are granted once, up front):

| Where | Pre-authorization | Removes the prompt for |
|-------|-------------------|------------------------|
| Bot App Registration → Expose an API | Teams Desktop / Teams Web / Copilot clients on `access_as_user` | **Teams → Bot** (Token A) |
| App-OBO → Expose an API | The **Bot** app on `access_as_user` | **Bot → App-OBO** (Token C) |

## Acceptance criteria — how you know it worked

Decoding the two tokens the bot holds shows exactly these claims (full detail in [Chapter 10](10-wiring-and-verification.md)):

| Token | Claim | Expected value | Confirms |
|-------|-------|----------------|----------|
| **Token C** | `aud` | `api://app-obo/3a0fad96-…` | Points at App-OBO, not `ai.azure.com` |
| | `scp` | `access_as_user` | It is a delegated (user) token |
| | `name` / `oid` | the real Teams user | The user's identity is carried |
| | `appid` | `2486b5cf-…` | Requested by the Bot app |
| **Token B** | `aud` | `https://ai.azure.com` | Foundry data-plane audience |
| | `appid` | `b0cc68f2-…` | The Foundry-access service principal |
| | `idtyp` | `app` | It is app-only (no user identity) |

Downstream, **Microsoft Graph receives Token D**, minted for the **original Teams user** — the same identity that started the conversation in Teams. The requirement is satisfied.

## Where to go next

- To understand **why** the default publishing path cannot do this → [01 — Overview & the Challenge](01-overview-and-challenge.md)
- To learn the **vocabulary** used everywhere below → [06 — The Five-Token Model](06-five-token-model.md)
- To **build it**, follow chapters [03](03-token-a-teams-sso.md) → [10](10-wiring-and-verification.md) in order.
