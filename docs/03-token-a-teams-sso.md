# 03 — Token A: Teams SSO (entry into the Bot)

## 3.1 The user does not call the agent — they call the Bot

When the user, from Teams, "invokes the agent", they are in fact invoking what the developer **registered in the Teams app manifest**: not Foundry's hosted-agent endpoint (`.../agents/<name>/endpoint/protocols/responses...`), but the **Azure Bot** that acts as the bridge. It is the **Bot** that receives the activity.

## 3.2 Token A — the Teams SSO token

To talk to the Bot **on behalf of the user**, Teams asks Entra to issue **Token A**, aligned with the *Expose an API* section of the registered application the Azure Bot was created with:

- **`aud = api://botid-2486b5cf-28b0-4f2d-b7c8-ff71aa856b72`** → corresponds to the **Application ID URI** exposed by the Bot's registered app.
- **`scp = access_as_user`** → corresponds to the **Scope** published (in *Expose an API*) by the same registered app.

In that same portal window, at the bottom, the applications **authorized** to obtain the token for this registered application are also listed — namely:

- Teams Desktop
- Teams Web

(How to configure the scope and these authorized clients is shown in [Chapter 04 §4.1](04-token-b-foundry-access.md#41-approach-1--token-b-as-a-user-delegated-token-not-recommended), steps 1A and 1B, since those settings are shared with the Token B configuration.)

## 3.3 What Token A is

Token A is therefore a **user-delegated** token destined for the Bot's app: it is the token that **enters the Bot**.

> This is an **SSO acquisition, not an OBO**. The OBO exchanges happen later — inside the bot (Token C) and inside the agent (Token D).

---

**Next:** [04 — Token B: Foundry Access](04-token-b-foundry-access.md)
