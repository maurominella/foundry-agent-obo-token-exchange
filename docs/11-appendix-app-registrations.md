# Appendix A — The Three Registered Applications

This appendix documents the three Entra registered applications used in this scenario — **one page per application**. Use the summary below to open each one.

## The three applications at a glance

| | [Bot](11a-app-bot.md) | [Foundry Access](11b-app-foundry-access.md) | [Downstream (App-OBO)](11c-app-downstream-obo.md) |
|---|---|---|---|
| **Display name** | `echo-token-bot` | `svc-foundry-dataplane-access-dev` | `svc-agent-obo-downstream-dev` |
| **Application (client) ID** | `2486b5cf-28b0-4f2d-b7c8-ff71aa856b72` | `b0cc68f2-87d7-491d-8cc2-60624256126e` | `3a0fad96-b026-4f5f-914a-fc6348656f6b` |
| **Purpose** | Creates and authenticates access to the bot service | Service identity that authenticates toward Foundry (mints **Token B**) | Service identity that performs OBO toward downstream services, e.g. MS Graph (mints **Token D**) |
| **Secret is used for** | OBO: **Token A → Token C** (via the OAuth Connection) | Client credentials → **Token B** (`aud = https://ai.azure.com`) | OBO: **Token C → Token D** (`aud = Microsoft Graph`) |
| **Open the page** | **[→ Bot](11a-app-bot.md)** | **[→ Foundry Access](11b-app-foundry-access.md)** | **[→ Downstream](11c-app-downstream-obo.md)** |

> **Directory (tenant) ID** for all three: `3ad0b905-34ab-4116-93d9-c1dcc2d35af6`.
> Values are from the validated development environment; secrets are masked. Substitute your own when reproducing.

## Who uses these applications?

- **Azure**, when it creates the Azure Bot.
- **The developer** (using the M365 Agents SDK), when creating the manifest.
- **Teams**, when it asks Entra for the user **Token A** to access the Bot.
- **The OAuth Connection**, to perform OBO and create the downstream user **Token C**.
- **The Bot service**, when it creates the Foundry app **Token B** to access Foundry.
- **The Foundry Hosted Agent**, when it performs OBO to obtain the MS Graph user **Token D**.

---

**Back to:** [README / Introduction](../README.md) · [Final Result](00-final-result.md)
