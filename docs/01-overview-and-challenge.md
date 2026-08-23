# 01 — Overview & the Challenge

## 1.1 The entry point: the Bot Service

The entry point is the **Bot Service**, the cloud router that allows Teams to communicate with the bot through the Bot Framework **Activity Protocol**.
The object we create in Azure — the **Azure Bot** — is the registration that connects the channels (Teams, Outlook, WhatsApp…) to our **custom bot** through the Bot Service.
In the Azure Bot's **Configuration** blade we set the **Messaging endpoint**, typically a public URL exposing `/api/messages`, where our **custom bot** is hosted.

Our custom bot receives *activities* from the Bot Service and produces replies, possibly invoking agents or external services. Those agents may in turn depend on **downstream services** — ticketing, databases, CRM — that need to know the identity of the user who asked the question.

For this to be possible, the identity has to be carried across three hops:

1. From **Teams** to the **bot**
2. From the **bot** to the **agent**
3. From the **agent** to the **downstream services**

Identity travels inside tokens whose claims include the **audience** (`aud`), which identifies the recipient of the token. Each hop therefore needs a token with a different audience. This chain of conversions — **token exchange** — is what lets each component operate **On Behalf Of the user (OBO)**.

## 1.2 Publishing the agent creates a closed Activity Protocol endpoint

When we publish the agent to the Agent 365 registry through the Foundry portal, an **Activity Protocol Endpoint** is created automatically, in the form:

```text
https://<foundry-account>.services.ai.azure.com/api/projects/<foundry-project>/agents/<agent-name>/endpoint/protocols/activityprotocol
```

By default Foundry sets this as the **Messaging endpoint** of the Azure Bot (in the *Configuration* blade).

This decoupling is exactly what lets us **replace** Foundry's Activity Protocol endpoint with the endpoint of **our own bot**, so we can implement custom behaviours that Foundry's Activity Protocol endpoint would not have provided. This custom bot is exposed over HTTP through an endpoint that typically ends with `/api/messages`.

For testing and troubleshooting, this also lets us run the custom bot **locally** — even launching it in *debug* mode from the development environment — and expose it to the Azure Bot through a **tunnel endpoint**, for example:

```text
https://hpgj8rcs-3978.euw.devtunnels.ms/api/messages
```

At the same time, the messaging endpoint can point to a private IP address, optionally intermediated by a gateway such as Azure Application Gateway or API Management (APIM).

## 1.3 The problem the custom bot solves

The custom bot addresses a problem inherent in the Activity Protocol endpoint created by Foundry: the **forwarding to Foundry of the authentication token received by the bot**.

While the Activity Protocol is perfectly capable of carrying OAuth/SSO flows, the Activity Protocol endpoint **created and managed by Foundry** does not expose a mechanism to configure OAuth + token-exchange + forwarding of the Bearer to Foundry on top of it. In other words, what is missing is the **complete implementation** covering our requirement — not the capability of the protocol.

In a bot written by us, on the other hand, we have **full control** over what is executed when the bot is invoked by the Azure Bot as its Messaging Endpoint. Let's see how it works.

---

**Next:** [02 — Authentication Flow & Prerequisites](02-authentication-flow-and-prerequisites.md)
