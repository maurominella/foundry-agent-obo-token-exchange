# Appendix A.1 — Bot Registered Application (`echo-token-bot`)

[← Back to the three applications](11-appendix-app-registrations.md)

**Role:** authenticates access to the bot service.

| Field | Value |
|-------|-------|
| Display name | `echo-token-bot` |
| Application (client) ID | `2486b5cf-28b0-4f2d-b7c8-ff71aa856b72` |
| Directory (tenant) ID | `3ad0b905-34ab-4116-93d9-c1dcc2d35af6` |

## Certificates & secrets

![Bot — Certificates & secrets](images/11-01-bot-certificates-and-secrets.png)
<br/>
*Bot registered application — Certificates & secrets*

The `client_id` + `client_secret` of the Bot registered application are used to **perform OBO**: to exchange **Token A** (presented by Teams to the app) for **Token C** (with `aud = api://app-obo/…/access_as_user`).

How do we perform this OBO? We could do it through an MSAL Confidential Application (as Foundry will later do to exchange that token for one with `aud = MS Graph`), but here we choose a more **managed** method: we use an **OAuth Connection**, to which we provide `client_id` and `client_secret` so it exchanges the user-token received from Teams (Token A) for **another user-token still associated with the same Teams user** (delegated identity preserved), but with the `aud` + `scp` of the downstream OBO app → **Token C**:

![Bot — OAuth Connection that mints Token C](images/11-02-bot-oauth-connection-token-c.png)
<br/>
*Bot — OAuth Connection (AAD v2) that exchanges Token A for Token C*

Note that the OAuth Connection can perform the OBO precisely because **the Bot is the `aud` of Token A**. You must authenticate as that audience → which is why the **Bot's credentials** are needed.

The **Token Exchange URL** is the element that enables the bot's **silent Single Sign-On (SSO)** in Teams. Without it the connection would only do interactive login (card + browser + redirect); with it, it can perform the SSO exchange without asking the user anything. This value equals the **App ID URI** of the Bot registered application — here `api://botid-2486…` — i.e. the identifier of the resource/audience Teams must request the first SSO token for (Token A). By setting the Token Exchange URL, we tell the OAuth Connection: *"to obtain the token you may attempt an SSO exchange; there is no need to show the sign-in card."* In fact, the one that generates that token is Teams: Teams sees that URI and silently asks Entra for an SSO token with `aud = that URI` → **Token A** (`aud = api://botid-2486b5cf...`).

Inside the BOT, Token A transits only in the `invoke signin/tokenExchange`, handled by `handleTeamsSigninTokenExchange` (`bot.js`), where it is available as `query.token`. The Token Service uses it as the assertion for the OBO and produces **Token C**. It is Token C that we then retrieve in `callStep` via `stepContext.result.token` (the `userAssertion` variable), and that we forward to the agent in the custom header:

```js
async handleTeamsSigninTokenExchange(context, query) {
    // TEMPORARY LOG: raw Teams SSO token
    // (expected aud = api://bot-<appId>)
    try {
        const t = query?.token || context.activity?.value?.token;
        if (t) {
            const p = JSON.parse(Buffer.from(t.split('.')[1], 'base64').toString('utf8'));
            console.log('--- SSO TOKEN IN INGRESSO (Teams) ---');
            console.log('aud  :', p.aud);
            console.log('scp  :', p.scp);
            console.log('appid:', p.appid || p.azp);
            console.log('upn  :', p.upn || p.preferred_username);
        }
    } catch (e) {
        console.log('Unable to decode the SSO token:', e.message);
    }
    await this.dialog.run(context, this.dialogState);
}
```

Inside the BOT we retrieve this Token C (in the `stepContext` of `callStep`, from which we extract the token — called `userAssertion` — with `stepContext.result.token`):

```js
// 2) Retrieve Token C (user assertion) - mainDialog.js
async callStep(stepContext) {
    // === Retrieval of Token C (user assertion) via OAuthPrompt ===
    // The delegated user token arrives here (stepContext.result). It is the
    // "third token" (Token C, aud=api://app-obo/<App-OBO-clientid>)
    // that will be forwarded to the agent in a custom header
    // for the downstream OBO.
    const tokenResponse = stepContext.result;
    const userAssertion = tokenResponse.token; // Token C
```

![Bot — Token C retrieval](images/11-03-bot-token-c-retrieval.png)
<br/>
*Bot — retrieving Token C (the user assertion) in `callStep`*

## Expose an API

![Bot — Expose an API: Application ID URI](images/11-05-bot-expose-api-app-id-uri.png)
<br/>
*Bot — Expose an API: the Application ID URI*

The **Application ID URI** lets you "point" to this registered application from the outside — for example when the Teams App (through its manifest) must point to the BOT that acts as the bridge toward the agent. Note that the URL reported here contains the `botid-` prefix; even better, you can use the explicit syntax `api://app-bot/2486…`.

![Bot — Expose an API: Scopes and authorized client applications](images/11-06-bot-expose-api-scopes-and-clients.png)
<br/>
*Bot — Expose an API: the `access_as_user` scope and the authorized Teams clients*

- **Scopes** — we add one of type `access_as_user`, because this is the actual scope the Teams client will use.
- **Client applications** — these are the client applications that can present themselves to Entra with their own token (received when the user signed in in the morning) to request the token needed for the scope above. The two applications shown here are in fact the two registered applications of **Teams Desktop** and **Teams Web**.

## API Permissions

![Bot — API Permissions](images/11-09-bot-api-permissions.png)
<br/>
*Bot — API Permissions: delegated permission to the App-OBO app + admin consent*

Here we specify the **delegated permissions** granted to the app, so that it can exchange its "own" tokens (those with `aud = api://botid-2486…` sent by Teams) for tokens with the **same user** but the scope of further Entra-protected services.

Specifically, since the bot must generate a token for the OBO app `svc-agent-obo-downstream-dev` with scope `access_as_user`, we declare here the delegation toward that app and scope.

The **"Grant for &lt;tenant_name&gt;"** avoids the consent that would otherwise be requested from the user.

---

[← Back to the three applications](11-appendix-app-registrations.md)
