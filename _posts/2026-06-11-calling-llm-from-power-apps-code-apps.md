---
layout: post
title: "A Code App cannot keep a secret: the four supported ways to call an LLM from Power Apps Code Apps"
date: 2026-06-11T00:00:00.000Z
tags:
  - power-platform
  - architecture
  - agentic-ai
  - ai-builder
human-approved: true
---

Power Apps Code Apps exist so professional developers can bring their own front end code into Power Platform's governance boundary. And almost every serious Code App eventually grows an AI requirement. Summarize the case the service agent has open. Draft a reply from the record on screen. Classify a request the moment it arrives. Pull structured fields out of pasted text. These are small, well-bounded LLM calls, the kind that turn a decent app into one users stop complaining about, and from the developer's chair the obvious implementation is the direct one: app code calls the model, model returns text, render it.

That implementation is off the table, and not because the product is immature. The hosting model rules it out. Microsoft Learn is blunt: a published Code App is "hosted on a publicly accessible endpoint," and sensitive data should not be stored in the app ([system configuration](https://learn.microsoft.com/power-apps/developer/code-apps/system-limits-configuration)). Anyone who finds the URL can read every byte of your published bundle, so an API key embedded in client code is an API key published to the internet. That single fact kills the direct call and replaces "how do I call the model" with three better questions: where does the secret live, which meter bills the tokens, and which governance surface sees the traffic.

On those three questions, the platform has four supported answers as of June 2026.

## The pattern matrix


| Pattern                                                  | Where the secret lives                                | Execution shape                           | Meter                                                              | Governance surface                                        |
| -------------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------ | --------------------------------------------------------- |
| Custom connector to an Azure API fronting the model      | Server side: app settings or Key Vault behind the API | Synchronous, streaming possible           | Azure consumption (model tokens billed to your Azure subscription) | DLP policy on the custom connector; Entra OAuth           |
| Dataverse prompt column                                  | None in your hands: the platform runs the prompt      | Asynchronous, on record create or update  | AI Builder credits, same licensing model as AI Builder prompts     | Dataverse security roles; environment AI feature settings |
| Power Automate flow wrapping an AI Builder prompt action | None in your hands: the prompt is a platform resource | Asynchronous from the app's point of view | AI Builder credits plus flow licensing                             | DLP on every connector the flow touches                   |
| Copilot Studio agent                                     | Managed by the agent's configuration                  | Conversational, multi-turn                | Copilot Credits, metered consumption                               | Agent-level channels, authentication, and admin controls  |


Each row deserves a paragraph, because the differences are not cosmetic.

## Custom connector to your own API: the default

The connector path is the documented integration surface for Code Apps. The SDK generates typed service classes for each connection, callable directly from your TypeScript ([connect to data](https://learn.microsoft.com/power-apps/developer/code-apps/how-to/connect-to-data)). Put an Azure Function or Web API behind a custom connector, have it call Azure OpenAI or a Foundry deployment, and keep the key in Key Vault. You control the model, the prompt template, the output contract, and the timeout behavior. Admin consent suppression is documented for custom connectors using Entra OAuth, so users are not interrupted by consent dialogs ([code apps overview](https://learn.microsoft.com/power-apps/developer/code-apps/overview)).

This is the most work of the four patterns, and the only one that gives you everything: synchronous responses, streaming, model choice, and a clean DLP story. When the LLM call is core to the app, build this.

## Dataverse prompt column: the closest thing to a direct prompt

If the workload is "generate a value from a record," the platform already ships the path. A prompt column is a Dataverse column whose value an AI Builder prompt generates whenever the record is created or one of the prompt's input columns is updated ([prompt columns](https://learn.microsoft.com/power-apps/maker/data-platform/prompt-column)). The Code App writes a record through the Dataverse connector and reads the generated column back. The feature reached general availability on May 4, 2026.

Know the mechanics before committing. Execution is asynchronous by design, decoupled from the transaction, and the documentation is explicit that on-demand execution is not supported: nothing recalculates until an input column changes. Each prompt column carries Status and Details companion columns with four states (NotStarted, InProgress, Completed, Failed), which is how your app knows when the value has landed. Input columns cannot be formula columns, file columns, images, or other prompt columns. And failures can be silent: if the environment's AI features are off or credits run out, execution is skipped without surfacing an error to your app.

That adds up to a specific shape: declarative, zero custom infrastructure, column-shaped output, eventual consistency. For enrichment and classification it is the cheapest pattern on the board. For "user clicks a button and waits for an answer," it is the wrong tool.

## Power Automate flow: the middle ground

Code Apps can call flows, and a flow can wrap an AI Builder prompt action. You get prompt reuse and platform-managed credentials without standing up Azure infrastructure. You also get flow latency, flow licensing arithmetic, and a second ALM artifact to ship with the app. It is a reasonable middle ground when the team already lives in Power Automate and the call volume is modest.

## Copilot Studio: only when it is actually a conversation

A Copilot Studio agent is the right answer when the requirement is genuinely conversational: multi-turn, grounded in knowledge sources, with session state. It is the wrong answer as a plain prompt-execution proxy. You would be paying for an orchestrator, channels, and a dialog engine to do what a single HTTP call does, and you would inherit the knowledge-source security decisions I wrote about in [the security trimming post](https://jakubbjl.github.io/2026/06/10/security-trimming-copilot-studio-knowledge-sources.html) without needing any of them.

## The path that does not exist

The path developers actually want, a first-class prompt API in the Code Apps SDK, is not documented anywhere on Learn as of June 11, 2026. The platform has plenty of prompt surfaces: AI Builder prompts are invocable from canvas apps and flows, Power Fx has AI functions in low-code plug-ins, Dataverse has prompt columns. None of them is wired into the Code Apps SDK as a callable service. The gap is real, and given that Code Apps already generate typed services for 1,500+ connectors, it is a gap I would expect Microsoft to close. Until then, the matrix above is the whole menu.

Note what every row has in common. Each pattern is also a billing decision and a governance decision. The connector path lands on your Azure bill, the prompt column and flow paths burn AI Builder credits, and Copilot Studio meters its own consumption. The end of flat-rate AI is an argument I keep returning to, and this is what it looks like at the level of a single architecture choice. Pick a pattern and you have picked a meter.

## Caveats

- The prompt column round-trip latency under load is not documented. Test it against your tolerance before promising it to users; the status columns make that test easy to script.
- Licensing here moves quickly. Verify AI Builder credit consumption rates and flow licensing against current terms at design time, not from this post's date.
- The absence of a direct prompt API is a statement about documentation as of June 11, 2026. Absences expire. Check Learn before treating it as current.

## References

- [Code Apps: system configuration](https://learn.microsoft.com/power-apps/developer/code-apps/system-limits-configuration)
- [Code Apps: overview and managed platform capabilities](https://learn.microsoft.com/power-apps/developer/code-apps/overview)
- [Code Apps: connect to data](https://learn.microsoft.com/power-apps/developer/code-apps/how-to/connect-to-data)
- [Dataverse prompt columns](https://learn.microsoft.com/power-apps/maker/data-platform/prompt-column)
- [AI Builder prompts overview](https://learn.microsoft.com/ai-builder/prompts-overview)

A Code App cannot keep a secret. Every supported pattern is a different answer to where the secret goes instead.