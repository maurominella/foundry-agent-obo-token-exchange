# Appendix A.2 — Foundry Access Registered Application (`svc-foundry-dataplane-access-dev`)

[← Back to the three applications](11-appendix-app-registrations.md)

**Role:** service identity to authenticate toward Foundry; it mints the app-only **Token B**. (It can also be realised as a User-Assigned Managed Identity, in which case it is secretless.)

| Field | Value |
|-------|-------|
| Display name | `svc-foundry-dataplane-access-dev` |
| Application (client) ID | `b0cc68f2-87d7-491d-8cc2-60624256126e` |
| Directory (tenant) ID | `3ad0b905-34ab-4116-93d9-c1dcc2d35af6` |

## Certificates & secrets

The secret of the Foundry Access registered application **is NOT used to perform OBO**. This secret (together with the `client_id`) is used to create the **app-token B** for Foundry (where the app is precisely the service identity for Foundry), which is used as the Bearer token when invoking the Foundry agent's Responses API endpoint:

```js
// mainDialog.js
const foundryCredential = new ClientSecretCredential(
    process.env.FOUNDRY_ACCESS_TENANT_ID,
    process.env.FOUNDRY_ACCESS_CLIENT_ID,
    process.env.FOUNDRY_ACCESS_CLIENT_SECRET
);
const foundryToken = (await foundryCredential.getToken('https://ai.azure.com/.default')).token;
const { answer, user } = await callFoundry(foundryToken, userText, userAssertion);
```

```js
// foundry.js
// Token B (app-only) in Authorization;
// Foundry validates it for RBAC and strips it.
// Token C (user assertion) travels in the custom header
// "x-client-user-token", which Foundry forwards
// to the agent container as-is (no chunking needed).
const headers = {
    Authorization: `Bearer ${foundryToken}`,
    'Content-Type': 'application/json'
};
if (userAssertion) {
    headers['x-client-user-token'] = userAssertion;
    console.log(`user assertion in header x-client-user-token: Token C len=${userAssertion.length}`);
}
const res = await axios.post(
    url, // "http://localhost:8088/responses?api-version=v1"
    { input: text },
    { headers }
);
```

![Foundry Access — Certificates & secrets](images/11-04-foundry-access-certificates-and-secrets.png)
<br/>
*`svc-foundry-dataplane-access-dev` — Certificates & secrets*

Key properties of this identity:

- This token is **stripped** by Foundry — which is why there is no point in it being a user-token (that would have required the user to hold the **Foundry User** RBAC role on the Foundry project). The user's identity travels **separately** in Token C.
- Naturally, the **service principal** of this identity must be added to the **Foundry User** of the Foundry project.
- **No user** is bound to this identity — only **app-tokens** are generated for it.
- It **can be shared** across all bots that need to open a conversation toward Foundry.
- It carries `client_id` + `client_secret` to build the `foundryCredential` object.
- It allows creating **Token B** via `foundryCredential.getToken('https://ai.azure.com/.default')`.
- The audience `aud = https://ai.azure.com` lets the token be used in the `Authorization` header toward the Foundry Hosted Agent via the Responses API.
- Its service principal must be **Foundry User** (RBAC) on the Foundry project.
- It needs **no API permissions**, because Foundry verifies the token is Foundry User, checks its `aud`, lets the call through, and then discards the identity.
- It needs **neither** an App ID URI **nor** a Scope.

## Expose an API

![Foundry Access — Expose an API: not needed](images/11-07-foundry-access-expose-api-not-needed.png)
<br/>
*`svc-foundry-dataplane-access-dev` — Expose an API: neither App ID URI nor scope is needed*

Here **neither** the Application ID URI **nor** a scope is needed, because **no one "authenticates" against this registered application**. The bot uses `client_id` + `client_secret` to **impersonate** the application and request a token, in its own name, toward Foundry:

```js
const foundryCredential = new ClientSecretCredential(
    process.env.FOUNDRY_ACCESS_TENANT_ID,
    process.env.FOUNDRY_ACCESS_CLIENT_ID,
    process.env.FOUNDRY_ACCESS_CLIENT_SECRET
);
foundryCredential.getToken('https://ai.azure.com/.default');
```

## API Permissions

This identity needs **no API permissions**. Foundry only verifies that the token is *Foundry User* and carries the right `aud` (`https://ai.azure.com`), lets the call through, and then discards the identity — the user's identity travels separately in **Token C** (see the [Downstream (App-OBO)](11c-app-downstream-obo.md) page for the OBO that consumes it).

---

[← Back to the three applications](11-appendix-app-registrations.md)
