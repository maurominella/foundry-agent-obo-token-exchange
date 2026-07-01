# 05 — Two Identity Axes

**Why did we use two registered applications?** Because the architecture has **two independent identity axes**, governed by different rules — and they must not be confused.

## 5.1 Axis 1 — Conversational (the Bot)

- **Rigid 1:1 relationship:** 1 agent → 1 bot → 1 Bot App Registration.
- The Bot's App Registration is the **addressable identity** on the channel: Teams / Azure Bot use it to route messages, and it defines the audience of the Teams SSO (Token A: `aud = api://botid-…`, scope `access_as_user`).
- It is **not shareable** across multiple bots: sharing it would make channel routing ambiguous and make every bot's SSO audience identical, so you could no longer distinguish which bot/agent the user is invoking.

## 5.2 Axis 2 — Foundry access (signs Token B)

- **Flexible relationship:** the number of identities does not follow the number of agents, but the **permission / trust boundaries**.
- It is simply a **credential to call the Foundry backend** (not an addressable identity), so it can be shared across several bots/agents — a choice, not an obligation.
- Share it when the agents live in the same project and use the same downstream permissions; separate it (down to one per agent) when downstream permissions diverge, or when you need isolation / auditability.

## 5.3 The asymmetry to remember

> It is acceptable to use the **same Foundry-access identity** for several bots/agents, but it is **not realistic to use the same Bot App Registration** for several bots.

- The **Foundry-access identity** says *"with which permissions do I call the backend"* → shareable.
- The **Bot App Registration** says *"who is the bot"* → necessarily unique.

## 5.4 Roles of the two registered applications and secret management

| | **Bot App Registration** | **Foundry-access App Registration** (with service principal) |
|---|---|---|
| **Reason to exist** | Conversational identity: auth to the Bot Framework / Azure Bot channel + Teams SSO (App ID URI, `access_as_user`) | Foundry data-plane access identity: obtains/signs Token B (`aud = https://ai.azure.com`) and holds the Foundry User role |
| **Cardinality** | 1 per bot (never shared) | 1 per permission boundary (shareable across bots) |
| **Needs a secret?** | **Yes** — it is the `MicrosoftAppPassword`, used by the bot runtime to authenticate to the Bot Framework / Azure Bot | **It depends:** yes if you use an App Registration (client credentials to sign Token B); **no** if you use a Managed Identity (UAMI), which is secretless |

### Practical guidance

- The Bot app **always** has its own secret (`MicrosoftAppPassword`) and is 1:1 with the bot.
- The Foundry-access identity is best realised, in production on Azure, as a **User-Assigned Managed Identity**: it removes the secret and makes even the "one identity per agent" scenario cheap (least privilege with no rotation cost). A dedicated App Registration with a secret remains the alternative when you must run outside Azure or need a portable identity.

---

**Next:** [06 — The Five-Token Model](06-five-token-model.md)
