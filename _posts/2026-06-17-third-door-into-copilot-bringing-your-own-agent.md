---
layout: post
title: "The third door into Copilot: where bringing your own agent meets the governance boundary"
date: 2026-06-17T00:00:00.000Z
tags:
  - m365-copilot
  - agentic-ai
  - architecture
  - governance
human-approved: true
---

I recently watched a Microsoft session from the Copilot Acceleration team on connecting a custom-built AI assistant to Microsoft 365 Copilot through a proxy agent. The demo is clean: an organization already runs its own agent, wraps it in a thin layer, and the agent shows up inside Copilot and Teams without the backend being rebuilt. The framing is reuse your investment and gain Microsoft's security and compliance envelope around it.

That framing is right, and it is also where the architecture work starts rather than ends. The interesting question is not whether you can surface an existing agent in Copilot. The docs are clear that you can. The question is what the envelope actually covers once you do, because the answer decides whether the pattern is safe for a regulated tenant or just convenient for a demo.

## Three doors, not one

Most conversations about "building a Copilot agent" collapse into one decision: open Copilot Studio. That is one door. Microsoft documents three, and they are different commitments, not three flavors of the same thing.

- **Declarative.** You bring instructions, knowledge, and actions, and you ride Copilot's own orchestrator and models. Lowest effort, least control over the reasoning.
- **Custom engine, built in Copilot Studio or pro-code.** You bring your own orchestrator and models. Copilot Studio is the low-code route; the Microsoft 365 Agents SDK is the pro-code route, in C#, JavaScript, or Python.
- **Custom engine, fronting an agent you already have.** This is the proxy. You keep a backend that was never built for Microsoft 365, wrap it with the Agents SDK, and surface it through Copilot.

The session is about the third door. It is the least discussed and the most operationally loaded, because the thing on the other side of the door was designed to someone else's assumptions, and now it has to behave inside Microsoft's.

## What the proxy actually is

Strip the demo's framing and the mechanism is documented plainly. To bring an existing agent into Copilot, you wrap it with the Microsoft 365 Agents SDK and put Azure Bot Service between the Copilot channel and your code. Bot Service translates what the channel sends into activities your code understands; the SDK wrapper listens for those activities, and your existing agent runs inside the handler. If your backend is already in C#, JavaScript, or Python, Microsoft's own guidance is that you do not have to modify it significantly. You add the SDK, register an app, produce a manifest, and the agent becomes discoverable in Copilot.

So the proxy is real and it is supported. It is a transport-and-identity adapter, not a re-platforming. That is the part worth being precise about, because the value and the risk both live in the word "transport."

## The envelope covers the front door, not the room

The demo's promise is Microsoft's security, compliance, and governance around your agent. Read against the documentation, that promise is true at the boundary and silent past it. The wrapper gives you Microsoft's identity, transport, and surface. It does not pull your backend inside the Microsoft 365 trust boundary, because your backend is still your backend, hosted where you host it, processing what it processes. Three boundaries decide whether that distinction is benign or a problem.

**Identity scoping is delegated, and the grounding API will not let you cheat it.** The honest version of the proxy passes the user's identity through to whatever the agent reads, so the agent sees only what the user is allowed to see. If your backend grounds on Microsoft 365 content, the supported path is the Microsoft 365 Copilot Retrieval API, which returns text from SharePoint, OneDrive, and Copilot connectors that the calling user has access to, respecting tenant access controls and keeping data in place. The detail that matters for a proxy design: that API supports delegated permissions only. Application permissions are not supported. You cannot stand up an app-only service identity and have it read across the tenant on the user's behalf. That is a deliberate design choice on Microsoft's side, and it is a good one, but it constrains how an always-on or background variant of your agent can ground itself. If the agent needs to act when no user is present, the easy grounding route is closed to it by design, and you are back to building and securing your own index with your own permission trimming, which is the cost the Retrieval API was supposed to remove.

**The secret is supposed to disappear, so do not let it reappear.** The supported posture for the Bot Service resource is a federated credential against a managed identity, with no client secret. You can delete the secret on the bot registration and authenticate with the managed identity instead. This is the same principle behind the proxy's appeal: the layer that fronts your agent should hold no standing credential of its own. The failure mode is quiet. A team under time pressure provisions the bot, sees that a client secret works, and ships it, and now the front door to a system that reaches enterprise data carries a long-lived shared secret that nobody rotates. The secret-free path is documented and it is not the default a hurried setup falls into. Choosing it is a governance decision, not a convenience.

**The government-cloud gates are real and they are not symmetric.** This is where the proxy pattern stops being universal. Publishing an agent through the Microsoft 365 Agents Toolkit is not supported in Microsoft 365 Government tenants, stated plainly in the docs. The Retrieval API that would ground the agent in tenant content is available in the global service and not in US Government L4, US Government L5 (DOD), or the China cloud operated by 21Vianet. So the exact pattern the session demonstrates, wrap an existing backend and ground it through the Microsoft 365 stack, is a commercial-cloud pattern. In a sovereign or government tenant, both the publishing route and the grounding API can be unavailable, and the workaround is more custom code, not less. Any reference architecture that shows this pattern without naming the cloud it assumes is incomplete.

## Why this belongs in an architecture conversation, not a build log

The reason to treat the proxy as an architecture decision rather than a wiring task is that it inherits a question the backend never had to answer: whose identity is this acting as, and what can it see. A standalone custom agent that a team built for its own portal had its own answer, usually a service account and an index it controlled. The moment that agent is fronted into Copilot, the right answer changes to the calling user's identity, and the supported tools enforce that change. The teams that struggle with the proxy are not the ones who cannot wire the SDK. They are the ones who built a backend that assumed broad service-level access and now have to reconcile it with a per-user, delegated model. The proxy does not create that gap. It exposes one that was already there, which is what most platform moves do to the assumptions a system was built on.

That is also why the proxy is a governance enabler rather than a governance bypass, when it is built the documented way. A custom agent scattered across a portal, a Teams app, and a web widget is three integrations to govern. The same agent fronted through Copilot with delegated identity, a secret-free bot, and Retrieval-API grounding is one surface, with one identity model and one place where access is enforced. Consolidation is the win, and the win is only real if the three boundaries above are honored. Skip them and you have not consolidated governance, you have given a broadly-scoped backend a more convenient front door.

## A short decision list before you wrap

Run an existing agent through these before committing to the proxy pattern.

- **Which cloud is the tenant?** If it is GCC, GCC High, DOD, or 21Vianet, confirm that both Agents Toolkit publishing and the Retrieval API are available before you design around them. Assume neither until you have checked.
- **Whose identity does the agent act as once it is in Copilot?** If the answer is still a service account, the proxy has surfaced an access model that needs redesigning first.
- **Does the agent need to ground in Microsoft 365 content?** If yes and a user is always present, the Retrieval API keeps data in place under the user's permissions. If it must run with no user present, the delegated-only constraint means you own the index and the trimming.
- **Is the bot registration secret-free?** Federated credential against a managed identity is the supported posture. A client secret on the front door of an enterprise-data system is a finding waiting to happen.
- **What stays outside the envelope?** Be explicit that the backend's own hosting, logging, and data handling are not covered by Microsoft's transport-layer compliance. The envelope ends at the adapter.

## Caveats and limits

- This is a reading of current Microsoft Learn documentation against a third-party session summary. The session is a lead, not a source; every product claim here is grounded in the docs, and the docs move. Verify availability, permission models, and cloud coverage for your tenant at design time rather than from this post's date.
- The custom-engine and Retrieval-API surfaces are evolving, and some are preview. Treat the delegated-only and government-cloud constraints as current behavior to confirm, not as permanent guarantees.
- The proxy pattern is one of several integration routes. Publishing a Foundry agent directly to Microsoft 365, building natively in Copilot Studio, or building pro-code with the Agents SDK from the start are all valid, and the right choice depends on how much of the backend already exists and how much control you need.
- Bringing an existing agent into Copilot does not relieve the backend of its own responsible-AI, security, and compliance obligations. It changes the front door, not the room.

## References

- [Custom engine agents for Microsoft 365 overview](https://learn.microsoft.com/microsoft-365/copilot/extensibility/overview-custom-engine-agent) (the three development approaches; the Foundry-via-Agents-Toolkit proxy row; knowledge-source access via Graph and the Retrieval API)
- [Bring your agents into Microsoft 365 Copilot](https://learn.microsoft.com/microsoft-365/copilot/extensibility/bring-agents-to-copilot) (wrap an existing C#/JS/Python agent with the Agents SDK; Azure Bot Service between channel and code; app registration and manifest; token management to scope to the user's identity)
- [Build custom engine agents with Microsoft 365 Agents SDK](https://learn.microsoft.com/microsoft-365/copilot/extensibility/m365-agents-sdk) (when to use the SDK; the verbatim note that publishing via the Agents Toolkit is not supported in Microsoft 365 Government tenants)
- [Provision agent resources in Azure Bot Service using federated credentials](https://learn.microsoft.com/microsoft-365/agents-sdk/azure-bot-create-federated-credentials) (delete the client secret; authenticate the bot with a managed identity via a federated credential)
- [Overview of the Microsoft 365 Copilot Retrieval API](https://learn.microsoft.com/microsoft-365/copilot/extensibility/api/ai-services/retrieval/overview) (grounding in place under the user's permissions; no separate index to secure)
- [Retrieve grounding data](https://learn.microsoft.com/microsoft-365/copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval) (delegated work-or-school permissions only, application permissions not supported; cloud availability table excluding US Gov L4, L5/DOD, and 21Vianet)

The proxy is the most generous-sounding of the three doors: keep what you built, gain the platform. It is also the one that quietly asks the hardest question, because the agent on the other side was built before that question had to be answered. Wrap the agent the documented way and the answer is a single governed surface. Wrap it the convenient way and you have only moved the front door.