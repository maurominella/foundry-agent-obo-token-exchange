# 10 — Wiring Token C & Verification

## 10.1 Verifying the tokens at runtime (Step 4, continued)

When the silent token acquisition is configured correctly, the two tokens the bot holds are exactly what they should be — with claims coherent with the whole flow. (Decode the JWT payload — the middle segment — and inspect the claims.)

### Token C — the "user" token the bot forwards to the agent

> This is exactly the token to place in the `x-ms-user-assertion` header. It is the token the agent will use as `user_assertion` in its `acquire_token_on_behalf_of`.

| Claim | Value | Outcome |
|-------|-------|---------|
| `aud` | `api://app-obo/3a0fad96-…` | Points at App-OBO → correct: it is the audience of the `user_assertion` |
| `scp` | `access_as_user` | Delegated → correct: it is a "user" token |
| `name` / `oid` | `Mauro Minella` / `68c4bf1e…` | Real user → correct: it represents the Teams identity |
| `appid` | `2486b5cf…` | The Bot app → correct: the bot is the client that requested Token C |

### Token B — the "app-only" token toward Foundry

> This token contains no user identity. It only authenticates the bot toward Foundry (the Hosted Agent endpoint).

| Claim | Value | Outcome |
|-------|-------|---------|
| `aud` | `https://ai.azure.com` | Foundry audience → correct |
| `appid` | `b0cc68f2-…` | Foundry-access service principal → correct |
| `idtyp` | `app` | App-only → correct: no user identity |

## 10.2 Step 5 — Forwarding Token C in `foundry.js`

### 1) `.env` — a configurable header

```bash
FOUNDRY_USER_ASSERTION_HEADER=x-ms-user-assertion
```

The header name is **configurable**: you can change it on the agent side if Foundry forwards it under a different name, avoiding hard-coding in the bot's code and allowing quick tests with alternative headers.

> The header name is **configurable** via `FOUNDRY_USER_ASSERTION_HEADER`. Both ends must agree on it: the bot writes Token C into this header and the agent reads it from the same header — in this reference, **`x-ms-user-assertion`**.

### 2) `foundry.js` — Token B in `Authorization`, Token C in the custom header

The separation is exactly right:

- **Token B → `Authorization`**: `audience = https://ai.azure.com`, `idtyp = app`; it is the app-only token; Foundry validates it for RBAC and **strips** it (does not forward it to the agent).
- **Token C → custom header**: `audience = api://app-obo/...`, `scp = access_as_user`; it contains the Teams user's identity; it is the `user_assertion`; Foundry **forwards** it to the agent, which uses it in `acquire_token_on_behalf_of`.

The log added below is very useful to verify the header is actually sent.

### 3) `mainDialog.js` — passing Token C

The fact that `callStep` now passes `userAssertion` to `callFoundry` closes the loop:

```text
Teams → Bot → Token A
Bot → Token C
Bot → Foundry → custom header
Foundry → Hosted Agent → Token C → OBO → Token D
```

The flow is complete.

## 10.3 Step 6 — How we send Token C in the custom header

The flow splits into two phases.

### Phase 1 — Retrieve Token C inside the BOT

Token C is a **delegated** token, therefore:

- you do **not** obtain it with `ClientSecretCredential` (that produces only Token B → app-only);
- you **do** obtain it through the bot's **OAuth Connection** (the new connection configured with the `access_as_user` scope);
- with Teams SSO, the `OAuthPrompt` receives the token **silently**, without a prompt.

**The three changes in the bot:**

1. **Re-enable the `OAuthPrompt`** in the constructor, but point it at the new connection — no longer `connectionName` / `ai-foundry-sso`, but `OBO_CONNECTION_NAME` (from `.env`).
2. **Re-enable `promptStep`** in the waterfall. The waterfall goes back to two steps: `promptStep` → retrieves Token C; `callStep` → uses Token C + Token B.
3. **In `callStep`, read the delegated token:**

```js
const tokenResponse = stepContext.result;
const tokenC = tokenResponse.token;   // aud = api://app-obo/...
```

At this point you have both tokens:

- **Token B** → app-only, obtained from `foundryCredential.getToken(...)` → goes in the `Authorization` header.
- **Token C** → delegated, obtained from `tokenResponse.token` → goes in the custom header (user assertion).

### Phase 2 — Package Token C in the POST toward Foundry

Pass Token C to `callFoundry` as the third parameter:

```js
await callFoundry(foundryTokenB, text, tokenC);
```

And inside `callFoundry`, put it in the custom header that Foundry forwards to the agent — the correct, already-validated code:

```js
async function callFoundry(foundryToken, text, userAssertion) {
    // Token B (app-only) in Authorization; Foundry validates it for RBAC and strips it.
    const headers = {
        Authorization: `Bearer ${foundryToken}`,
        'Content-Type': 'application/json'
    };

    // Token C (user assertion) in the custom header; Foundry passes it through to
    // the agent, which uses it as the OBO assertion to mint downstream tokens.
    const assertionHeader = process.env.FOUNDRY_USER_ASSERTION_HEADER;
    if (userAssertion && assertionHeader) {
        headers[assertionHeader] = userAssertion;
        console.log('user-assertion header:', assertionHeader, '(Token C forwarded)');
    }

    const res = await axios.post(
        url,
        { input: text },
        { headers }
    );
}
```

### What happens next

- Foundry validates **Token B** (RBAC).
- Foundry does **not** forward Token B.
- Foundry **forwards Token C** to the agent.
- The agent uses Token C as the `user_assertion`.
- The agent performs the **OBO → Token D** (Graph, CRM, enterprise API…).
- **The flow is complete.**

## 10.4 🔥 Result: the bot now implements OBO end-to-end

We have:

| Token | Role |
|-------|------|
| **Token A** | from Teams |
| **Token B** | app-only, toward Foundry |
| **Token C** | delegated, toward App-OBO |
| **Token D** | delegated, toward the downstream service |

…and they are all routed the correct way. The signed-in Teams user's identity reaches Microsoft Graph — on the user's behalf — with **no login and no consent prompt** anywhere in the path.

---

**Back to:** [README / Introduction](../README.md) · [Final Result](00-final-result.md)
