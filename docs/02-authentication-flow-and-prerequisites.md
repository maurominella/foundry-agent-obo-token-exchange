# 02 — Authentication Flow & Prerequisites

This chapter sets up the vocabulary and the prerequisites for the whole flow — *from the "pre-token" to the invocation of Foundry.*

## 2.1 Prerequisites

**Prompt agent vs hosted agent.** A Foundry agent can take the form of a **prompt agent** — a declarative agent whose tool orchestration is provided by the Agent Service — or a **hosted agent**, where Foundry acts as a full hosting platform. For the goal of *authenticating the client-connected user to Foundry*, there is no difference between a prompt agent and a hosted agent. However, if the agent needs to **access a backend system authenticated with the identity of the connected user**, some code has to run that cannot do without a **hosted agent**.

> **MCP servers are out of scope** for this discussion: an MCP server follows a different implementation and authentication logic in Foundry (via *OAuth Identity Passthrough*) than the case examined here, where it is the **agent itself** — at most through Function Calling — that accesses the backend system, without using external services.

**The 1:1 relationships.** When the agent is published to the Agent 365 registry through Foundry, it gets a **1:1 relationship with the Bot**, which bridges the *Microsoft 365 first-party client* (Teams / Copilot or other Bot channels) and the agent published by Foundry on its infrastructure.

Another 1:1 relationship is the one between the **Bot** and its **registered application** — so called because it is managed through the *App registrations* blade of Entra. In some cases this application is predefined by Microsoft (e.g. Teams and Foundry); in others the registered application is created ad hoc for the specific service, like our custom bot.

## 2.2 The three App Registrations

To make this solution as secure and flexible as possible, three **App Registrations** are created in Entra:

| # | App Registration | Role | Shareable? |
|---|------------------|------|------------|
| 1 | **Bot-registered application** | The conversational identity of the bot. **One per Agent/Bot.** | No — one per bot |
| 2 | **Foundry-registered application** (`svc-foundry-dataplane-access-dev`) | Obtains access to the Foundry service. Not tied to a specific user. | Yes — can be shared across Agents/Bots |
| 3 | **Downstream-registered application** (`svc-agent-obo-downstream-dev`, "App-OBO") | Used to access the downstream services. Bound, case by case, to specific users and to the services declared in the app's own configuration. | Yes — can serve multiple agents and multiple downstream services |

Each of these is configured in detail in the chapters that follow.

## 2.3 Entra claims

The Identity Provider used in this scenario is **Microsoft Entra**, which generates authentication tokens carrying characteristics — called **claims** — such as:

- **`aud`** → the **audience**, i.e. the target service the token must be presented to in order to gain access. For example: Microsoft Teams, Foundry, the Bot Service, or Microsoft Graph through the (custom or Microsoft-predefined) registered application associated with the service.
- **`scp`** → the **scope** the token is built for, e.g. accessing files, users, or mail in the case of Microsoft Graph.
- **`idp`** → the identity of the user / service principal / managed identity Entra used to **sign** the token — in other words its owner.
- **`signature`** → every token is signed, to prevent man-in-the-middle risks.
- **`name`, `lastname`, `upn`, `address`…** → other claims that identify the user.

## 2.4 The tokens at a glance

**Pre-token — the user's authentication to Teams.** When the user connects to Teams, they establish an authenticated session towards Microsoft 365 / Teams: Entra has issued the tokens for the client session (we generically call this the **pre-token**, with `aud = Teams*`). This pre-token is **not** used directly to call the agent or the bot: it only represents the identity context from which — thanks to the SSO configuration declared in the Teams app manifest (`webApplicationInfo`) — the first useful token (**Token A**) can be requested.

The full flow then runs through four working tokens:

| Token | In one line |
|-------|-------------|
| **Token A** | Teams SSO token — the user's entry into the Bot. See [Chapter 03](03-token-a-teams-sso.md). |
| **Token B** | App-only token to get past Foundry's RBAC gate. See [Chapter 04](04-token-b-foundry-access.md). |
| **Token C** | The user assertion — the user's identity, carried in a custom header. See [Chapter 07](07-token-c-user-assertion.md). |
| **Token D** | The downstream token the agent mints via OBO. See [Chapter 08](08-token-d-downstream-obo.md). |

The full canonical table is in [Chapter 06 — The Five-Token Model](06-five-token-model.md).

---

**Next:** [03 — Token A: Teams SSO](03-token-a-teams-sso.md)
