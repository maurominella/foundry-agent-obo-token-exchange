# Appendix A.3 — Downstream Access Registered Application (`svc-agent-obo-downstream-dev`, "App-OBO")

[← Back to the three applications](11-appendix-app-registrations.md)

**Role:** service identity to perform OBO toward the downstream services (e.g. MS Graph) from inside the Foundry hosted agents; it mints **Token D**.

| Field | Value |
|-------|-------|
| Display name | `svc-agent-obo-downstream-dev` |
| Application (client) ID | `3a0fad96-b026-4f5f-914a-fc6348656f6b` |
| Directory (tenant) ID | `3ad0b905-34ab-4116-93d9-c1dcc2d35af6` |

## Certificates & secrets

This secret, together with the `client_id`, is used to **perform OBO**: we use as the `user_assertion` the Token C provided by the BOT, obtaining **Token D** with `aud = MS Graph` (= `https://graph.microsoft.com/Files.Read`).

To perform OBO it requires creating a **confidential application** through MSAL, with `client_id` and `client_secret`:

```python
CLIENT_USER_TOKEN_HEADER = "x-client-user-token"
GRAPH_SCOPES = ["https://graph.microsoft.com/Files.Read"]
assertion = context.client_headers.get(CLIENT_USER_TOKEN_HEADER, "")

conf_app = msal.ConfidentialClientApplication(
    client_id,
    client_credential=client_secret,
    authority=f"https://login.microsoftonline.com/{tenant_id}"
)
msgraph_token = conf_app.acquire_token_on_behalf_of(
    user_assertion=user_assertion, scopes=GRAPH_SCOPES
)
```

The full agent-side snippet, as it reads Token C from the custom header and exchanges it:

```python
# Token D: App-OBO (confidential client) exchanges
# Token C for a Graph token.
import msal  # On-Behalf-Of token exchange (C -> MS Graph)
GRAPH_SCOPES = ["https://graph.microsoft.com/Files.Read"]
user_assertion = context.client_headers.get(CLIENT_USER_TOKEN_HEADER, "")
app = msal.ConfidentialClientApplication(
    client_id,
    client_credential=client_secret,
    authority=f"https://login.microsoftonline.com/{tenant}",
)
result = app.acquire_token_on_behalf_of(
    user_assertion=user_assertion, scopes=GRAPH_SCOPES
)
```

Key properties of this identity:

- It **can be shared** across all bots/agents that need to reach downstream services (e.g. MS Graph).
- The downstream services must be **authorized** in the Entra **API Permissions** section (see below).
- It serves to create **user-tokens** in which the user is extracted from the "source" token being exchanged.

## Is `x-client-user-token` a "fake header"?

No — it is a **documented Foundry mechanism**: `x-client-*` is the prefix for **pass-through client headers** (Request direction, i.e. from the client to the agent). The exact same construct exists in the Python package (`azure-ai-agentserver-core`, `PlatformHeaders` / `CLIENT_HEADER_PREFIX = "x-client-"`).

In other words, the gateway recognizes the `x-client-*` prefix as the **official pass-through channel** toward the agent container (in the `azure-ai-agentserver-core` SDK, the constant `CLIENT_HEADER_PREFIX = "x-client-"`). The agent reads it via `context.client_headers`. So it is exactly the channel intended to carry context/credentials from the client to the agent.

**Fundamental point: the security of Token C does not depend on which header carries it, but on its cryptographic properties:**

- It is a **JWT signed by Entra (RS256)**. It cannot be forged: no one can fabricate a valid Token C without Entra's private key.
- It is **audience-restricted** (`aud = api://app-obo/...`): usable only toward that downstream app, nowhere else.
- It is **short-lived** (~1h).
- To "spend" it (OBO → Graph) you need the **App-OBO `client_secret`**: merely holding the token is not enough. Indeed, with a malformed/forged token, Entra responds `AADSTS50027` and the OBO fails.
- **Consequence:** even if someone intercepted or injected a value into that header, they would get nothing — the agent does not blindly "trust" the header, because the token is cryptographically validated by Entra at OBO time.

**In transit it is as protected as the `Authorization` header.** Calls to Foundry are over HTTPS/TLS: all headers (including `x-client-user-token`) are encrypted in transit, exactly like `Authorization`. There is no protection difference between a standard header and a custom header over TLS. The body would be encrypted the same way too — so "header vs body" is irrelevant for confidentiality in transit.

> For the official Microsoft references backing this up, see [Appendix B — Microsoft Official Documentation](12-appendix-microsoft-documentation.md).

## Expose an API

![App-OBO — Expose an API: scope and authorized client](images/11-08-app-obo-expose-api-scope-and-client.png)
<br/>
*`svc-agent-obo-downstream-dev` — Expose an API: the `access_as_user` scope and the authorized client (the Bot)*

This scope is used when the bot creates the token to pass to Foundry in the `x-client-user-token` parameter — associated with the user connected to Teams (which the bot finds in the token it received from Teams) and destined for this downstream OBO app.

The **authorized client application** shown here is the application authorized to request tokens for this downstream OBO app; as noted, the token is requested by the **bot**, using the identity indicated in the user token it received from Teams — i.e. the **Bot Registered Application**.

## API Permissions

![App-OBO — API Permissions: Microsoft Graph Files.Read](images/11-10-app-obo-api-permissions-graph-files-read.png)
<br/>
*`svc-agent-obo-downstream-dev` — API Permissions: Microsoft Graph `Files.Read` (delegated)*

![App-OBO — API Permissions: admin consent granted](images/11-11-app-obo-api-permissions-admin-consent.png)
<br/>
*`svc-agent-obo-downstream-dev` — API Permissions: admin consent granted*

Here we specify the **delegated permissions** granted to the app, so that it can exchange its "own" tokens (those with `aud = api://app-obo/3a0f…`) for tokens with the **same user** but the scope of further Entra-protected services.

Specifically, since the hosted agent must "exchange" the token received from the bot through the `x-client-user-token` header — which has `aud = api://app-obo/3a0f…` and `user = <Teams_user>` — for a token with `aud = MS Graph` and `scp = Files.Read`, it needs the `client_id` + `client_secret` of the same downstream OBO app.

The **"Grant for &lt;tenant_name&gt;"** avoids the consent that would otherwise be requested from the user.

---

[← Back to the three applications](11-appendix-app-registrations.md)
