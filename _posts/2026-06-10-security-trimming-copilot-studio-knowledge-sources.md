---
layout: post
title: "Security trimming is a knowledge-source decision in Copilot Studio"
date: 2026-06-10 09:00:00 +0200
tags: [copilot-studio, agentic-ai, governance, security, microsoft, power-platform]
human-approved: true
---

Two agents can return the same answer in a demo and carry very different exposure in production. One shows the user only what that user is allowed to read. The other answers from everything in the index, including records the person asking was never cleared to see. The answer text looks identical. The governance is not.

The variable that separates them is security trimming, and on the Microsoft stack it is not a switch you turn on. It is a property of each knowledge source you attach to the agent. Get the sources right and the permission model holds end to end. Get them wrong and the agent quietly routes around the access controls you spent years building.

I have written that an agent's value and its safety are set by its retrieval architecture, not by its model and not by data cleanliness alone. This is the part of that argument the LinkedIn feed has no room for: the actual map of which Copilot Studio knowledge sources preserve a user's permissions, which do not, and what to do about the one most teams reach for first.

## What security trimming means, and where it happens

Security trimming is the retrieval-time check that returns only the content the asking user has permission to read. The user asks a question, the agent searches, and before any result reaches the model, the system filters it against that specific user's access rights. No trimming means the filter is absent: the agent retrieves from the whole index and summarizes whatever ranks highest, regardless of who is asking.

Copilot Studio runs retrieval in a fixed sequence. It rewrites the user's question for better matching, runs the rewritten query against every knowledge source you configured, takes the top results from each, and then summarizes with citations. The trimming, if it happens at all, happens during that retrieval step, and it depends on how each source authenticates. That is the detail that decides your exposure, and it is invisible in a demo because the maker testing the agent usually has access to everything anyway.

## The map: which sources are delegated

A source is delegated when retrieval runs under the signed-in user's Microsoft Entra identity, so the agent inherits that user's permissions. Microsoft documents the behavior of each source directly. The table below is drawn from the Copilot Studio retrieval guidance, current as of June 2026.

| Knowledge source | Authentication | Per-user security trimming |
| --- | --- | --- |
| SharePoint / OneDrive | Microsoft Entra ID, delegated | Yes. Returns only content the user has read access to. |
| Dataverse tables | Microsoft Entra ID, delegated | Yes. Retrieval runs under the user's identity. |
| Graph connectors (ServiceNow, Confluence, and similar) | Microsoft Entra ID, delegated | Yes. Honors the user's permissions in the indexed source. |
| Real-time connectors (Salesforce, ServiceNow, Zendesk, Azure SQL) | User must be logged in | Yes, scoped to the user's own connection and credentials. |
| Azure AI Search | Configured endpoint | No. The connection is not delegated. No per-user check. |
| Public websites (Bing) | None | Not applicable. Public content only. |
| Uploaded files (stored in Dataverse) | None | No. Anyone who can use the agent can retrieve from them. |
| Custom data (via flows, connectors, HTTP) | None at the source | Only what your upstream query enforces before it hands data back. |

Read the table as a design constraint, not a ranking. The delegated sources carry your permission model for free. The rest do not, and three of them deserve a closer look because teams reach for them without registering the tradeoff.

## The Azure AI Search trap, stated precisely

Azure AI Search is a strong retrieval engine and the right choice for many agents. It is also the single source most likely to break your permission model silently, and the reason is subtle enough that it survives most reviews.

Microsoft's own documentation is blunt about the Copilot Studio connection: it is not delegated, which means no security trimming and no authentication requirement for the user. The agent queries the index as a service, retrieves the top matches, and summarizes them. It never asks who is sitting in front of it.

Here is the part that catches experienced teams. Azure AI Search the service can enforce document-level access control. There are now several supported approaches: security filters that match a caller identity string at query time, and newer methods covering storage ACLs, SharePoint permissions, and Microsoft Purview sensitivity labels, several of them in preview on the 2026-05-01 API. So a reasonable architect concludes the index is secured and moves on.

The gap is that the Copilot Studio knowledge-source connection does not pass the end user's identity to the index. Document-level access control in Azure AI Search is enforced against the caller's identity at query time. If the caller is a service connection with no per-user identity, there is no identity to enforce against. The index can be perfectly capable of trimming and still return everything, because nothing is telling it whom to trim for.

So the correct statement is narrow and it matters. The capability lives in the service. The enforcement requires a per-user identity at query time. The Copilot Studio connection does not supply one. Securing the index does not, by itself, give you per-user trimming inside the agent.

If per-user trimming is a requirement and you still want Azure AI Search, you have three honest options. Scope the index so its entire contents are safe for everyone who can reach the agent, and treat the agent's audience as the security boundary. Or restrict who can use the agent so the audience matches the data, which turns access control into an agent-sharing problem rather than a retrieval problem. Or use a delegated source, SharePoint, Dataverse, or a Graph connector, for the content that genuinely needs per-user permissions, and reserve Azure AI Search for content that does not. What you cannot do is assume the index's own security features are reaching the user. They are not on this path.

## The maker default that compounds it

Retrieval is only half the exposure. The other half is who is allowed to talk to the agent at all.

A maker can set an agent's authentication to "No authentication," and Microsoft is clear about what that means: anyone who has the link can chat with it, and you lose any ability to control which users in your organization can use it. Pair that default with a non-delegated knowledge source and the failure compounds. An unauthenticated agent, answering from an untrimmed index, will hand any anonymous visitor whatever the index contains. Neither setting is a product defect. Each is a choice a maker can make without anyone noticing, which is exactly why it cannot be left to chance.

This is where governance earns its keep, and where guardrails enable speed rather than slowing it. A tenant data policy in the Power Platform admin center can require authentication, which removes the "No authentication" option from every maker in scope. The same policy surface can block specific knowledge sources and connectors outright. Set those controls once at the tenant level and makers are free to build quickly inside a boundary that holds, instead of each agent becoming a separate security review.

## A pre-ship acceptance test

Architecture is a discipline, not a setting, so make the discipline concrete. This is the check I run on an agent before it ships. It takes minutes and it catches the failures that otherwise surface in front of users.

1. **List every knowledge source.** For each one, write down its authentication model from the table above. If you cannot name the model, you are not ready to ship.
2. **Flag every non-delegated source.** For each, answer one question in writing: what stops a user from retrieving content they are not entitled to read? "The index is secured" is not an answer unless you can show the per-user identity reaches the query.
3. **Name the security boundary for each non-delegated source.** It is either the index scope (everything in it is safe for all agent users) or the agent's audience (sharing is locked to people cleared for the data). One of those must be true and documented.
4. **Confirm authentication is required centrally.** Verify a tenant data policy requires authentication, so no maker can switch it off. Do not rely on the agent-level setting alone.
5. **Test as a low-privilege user, not as the maker.** Sign in as someone with minimal access and ask the agent for something sensitive. The maker's own access hides exactly the failure you are testing for.
6. **Turn on audit before go-live.** You want a record from the first interaction, not from the first incident.

Six steps, and step two is where most of the value is. A non-delegated source is not forbidden. It is a decision that has to be made on purpose, with a compensating control written down, rather than stumbled into because the demo looked fine.

## Caveats

This describes documented product behavior as of June 2026. The Azure AI Search document-level access methods are moving quickly and several are in preview, so verify the current state for your configuration before you rely on it. Security trimming is a read-time permission check; it is not a substitute for fixing oversharing and broken ownership at the source, which is a separate and equally real problem. And none of this is an argument against Azure AI Search, which remains the right engine for a large class of agents. The argument is narrower: choose each knowledge source for its security model as deliberately as you choose it for its answer quality, because in production those are the same decision.

The model you can swap in an afternoon. The retrieval architecture and the identity model are what you actually own. Build those on purpose.

---

*Sources, all Microsoft Learn: [Enhance AI responses by using Retrieval Augmented Generation](https://learn.microsoft.com/microsoft-copilot-studio/guidance/retrieval-augmented-generation) (knowledge-source authentication and trimming table); [Document-level access control in Azure AI Search](https://learn.microsoft.com/azure/search/search-document-level-access-overview) (security filters, ACL/RBAC, SharePoint ACLs, Purview labels); [Configure user authentication in Copilot Studio](https://learn.microsoft.com/microsoft-copilot-studio/configuration-end-user-authentication) (No authentication, Authenticate with Microsoft); [Implement a zoned governance strategy](https://learn.microsoft.com/microsoft-copilot-studio/guidance/sec-gov-phase2) (tenant data policies for authentication, knowledge sources, and connectors).*
