# Appendix B — Microsoft Official Documentation

This appendix collects the **official Microsoft sources** that back the design choices in this documentation — in particular the use of the `x-client-*` pass-through headers and the On-Behalf-Of support in Foundry hosted agents.

## Summary table

| # | What it states | Microsoft source (page / section) | Exact excerpt |
|---|----------------|-----------------------------------|---------------|
| **1** | The `x-client-` prefix is an official **client → agent pass-through** channel | [Azure AI Agent Server Core library for .NET](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.agentserver.core-readme?view=azure-dotnet-preview) — *PlatformHeaders* section | `ClientHeaderPrefix` → `x-client-*` → *Request* → "Pass-through client header prefix" |
| **2a** | Hosted agents support **OBO with a user token** | [Hosted agents in Foundry Agent Service](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents) — *Agent identity and endpoint* section | "User-invoked scenarios (interactive): If a user token is present, the platform supports OAuth 2.0 On-Behalf-Of (OBO) flows. In this case, the agent can call downstream services on behalf of the user using the user's delegated permissions, subject to Microsoft Entra ID tenant policies." |
| **2b** | The **"attended" OBO model** described in detail | [Agent identity concepts in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity) — *Authentication capabilities* section | "Attended (delegated access or on-behalf-of flow): The agent operates on behalf of a human user, using the OAuth 2.0 on-behalf-of (OBO) flow. The user first authenticates to the application, and the application passes the user's token to Agent Service…" |
| **3** | `x-client-*` headers are treated as **pass-through** by the Foundry platform | [GitHub issue Azure/azure-sdk-for-python#45797](https://github.com/Azure/azure-sdk-for-python/issues/45797) — answered by a Microsoft engineer | Customer-reported; Microsoft reply: "All x-client-* headers are treated as pass through by the platform" — this does **not** include the OAuth token provided in the `Authorization` header |

## Detailed description

### 1. The `x-client-` prefix is an official pass-through channel — Microsoft Learn (.NET SDK reference)

Page: [Azure AI Agent Server Core library for .NET](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.agentserver.core-readme?view=azure-dotnet-preview) → *PlatformHeaders* section. The official table explicitly lists the constant `ClientHeaderPrefix = x-client-`, direction *Request*, description "Pass-through client header prefix". This is official documentation on `learn.microsoft.com`: it states, in black and white, that `x-client-` is the prefix for client headers forwarded from the client to the agent. The exact same construct exists in the Python package (`azure-ai-agentserver-core`, `PlatformHeaders` / `CLIENT_HEADER_PREFIX = "x-client-"`).

### 2. Hosted agents support OBO with a user token — Microsoft Learn (concept pages)

Two concept pages confirm it:

- **[Hosted agents in Foundry Agent Service](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agents)** (*Agent identity and endpoint*): if a user token is present, the platform supports the OAuth 2.0 On-Behalf-Of flow, and the agent can call downstream services with the user's delegated permissions.
- **[Agent identity concepts in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)** (*Authentication capabilities*): describes the "attended (OBO)" model in which the application passes the user token to Agent Service, which exchanges it for a token carrying the agent's identity and the user's delegated permissions.

### 3. "Foundry strips the token" — there is NO official Microsoft documentation

An important and honest clarification: there is **no** Microsoft Learn page stating that Foundry "strips" the `Authorization` header (or the user identity) after using it for authentication. The only public evidence is the GitHub issue **[Azure/azure-sdk-for-python#45797](https://github.com/Azure/azure-sdk-for-python/issues/45797)** (a customer report, with input from Microsoft employees), where:

- it is observed that the `Authorization` header is **not** propagated to the agent container (with the `from_agent_framework` adapter);
- a Microsoft member (RaviPidaparthi) recommends precisely the `x-client-*` headers as the transport channel: *"All x-client-* headers are treated as pass through by the platform."*

---

**Back to:** [README / Introduction](../README.md) · [Appendix A](11-appendix-app-registrations.md)
