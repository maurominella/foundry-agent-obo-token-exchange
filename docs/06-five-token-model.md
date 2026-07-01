# 06 — The Five-Token Model

To reason about this architecture without confusion, every token is given a **canonical name**. From here on, the documentation uses only these names.

## 6.1 The canonical token table

| Name | Issued by → for | `aud` | How it is obtained | Where it travels |
|------|-----------------|-------|--------------------|------------------|
| **Pre-token** | Entra → the user at Teams login | Teams / MS | Interactive Teams login | Stays inside the Teams client |
| **Token A** | Entra → Teams, for the Bot | `api://botid-<BotApp>` (`scp = access_as_user`) | Teams SSO | Teams → Bot (sign-in / token exchange) |
| **Token B** | Entra → Bot (app-only) | `https://ai.azure.com` | Client credentials of the dedicated SP | Bot → Foundry, `Authorization` header (stripped after RBAC) |
| **Token C** (the "third token" / user assertion) | Entra → Bot, via token exchange from Token A | `api://<App-OBO>` (`scp = access_as_user`) | Bot token exchange (OAuthPrompt / connection with scope = App-OBO) | Bot → Foundry, **custom header** → agent |
| **Token D** (downstream) | Entra → Agent, via OBO from the third token | Graph / Fabric / a 3rd service | Agent's OBO exchange (using the App-OBO secret) | Agent → downstream API |

## 6.2 Two notes that complete the picture

**Token B is a side branch — it is not part of the OBO chain.** It is only the app token for the `Authorization` header. The delegated chain is:

```text
Pre-token  →  Token A  →  Token C  →  Token D
```

**The third token (Token C) is a brand-new token**, derived from Token A through the bot's token exchange, with `aud = App-OBO`. It is **not** Token A forwarded as-is — because Token A has `aud = api://botid-<Bot>`, the wrong audience for the agent's OBO step.

---

**Next:** [07 — Token C: The User Assertion](07-token-c-user-assertion.md)
