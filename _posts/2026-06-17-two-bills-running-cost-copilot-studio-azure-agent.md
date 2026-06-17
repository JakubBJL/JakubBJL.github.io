---
layout: post
title: "Two bills, not one: the running cost of a Copilot Studio and Azure agent"
date: 2026-06-17T00:00:00.000Z
tags:
  - copilot-studio
  - azure
  - finops
  - architecture
human-approved: true
---

Before a client will scale a pilot, someone asks the question the demo never answers: what does this cost to run. The honest answer starts by refusing the premise of a single number. An agent built on Copilot Studio with an Azure retrieval and reasoning backend does not have a bill. It has two, and they behave so differently that adding them together is the first mistake most cost conversations make.

One bill is Azure consumption: storage, search, the model calls, the compute that runs retrieval. It is metered, it scales with use, and at a pilot it is small. The other bill is licensing: the Power Platform and Microsoft 365 seats that the people and the automations need, plus the agent's own capacity. It is charged per user and per month, it barely moves with use, and at a pilot it is most of the money. The two land on different invoices, bill on different logic, and grow in opposite directions. Treat them as one line and you will misjudge both the pilot and the production cost, each in the wrong direction.

What follows is a scoping discipline, not a price list. The numbers age; the structure does not. Every figure here is list price, and Microsoft and Azure list prices change. The reusable part is which questions to ask and which lines to watch.

## The first invoice lies

The most useful thing I can say about an agent's first Azure invoice is: do not trust it. On one build, the first invoice carried a single line, the search tier. The heavy model calls had run after the billing cutoff, so the bill showed the floor and nothing else. Read literally, it said the agent was nearly free. Read correctly, it said the always-on cost was a fixed search tier and the variable cost had not landed yet.

That gap, between what an invoice measured and what the running cost will be, is where forecasts go wrong. So separate the two honestly. The measured cost is what an invoice and an Azure Cost Management export actually show. The modeled cost is everything else: a projection from current unit prices and explicit assumptions about volume, and no more solid than that until a real month of usage reconciles it. Say which is which every time you put a number in front of a budget owner. A model presented as a measurement is how a proposal earns a surprise in month two.

## Bill one: licensing, the floor that recurs

At pilot scale, licensing is where the ongoing money sits, because it recurs every month no matter how often the agent runs. Three groups need it: the people who use the app, the identity that runs the automations, and the agent itself. Each carries a dependency that stays invisible until it lands on an invoice or blocks a deployment.

The people who use a model-driven app on custom Dataverse tables each need a premium Power Apps license. That is the seat, and at pilot volume the seats are the bill.

The agent's access design carries its own dependency. Restricting an agent to a security group through an Entra enterprise-application assignment with assignment required set to yes is a group-based assignment, and group-based assignment to an application requires Microsoft Entra ID P1 or higher. The access pattern you chose for sound security reasons is also a tenant-level licensing commitment. It is a real cost of the design, not an optional extra.

The agent's usage is metered in Copilot Credits, not messages, since September 2025. Internal users who already hold a Microsoft 365 Copilot license incur no credit charge for employee-facing use, within fair-use limits that Microsoft reserves the right to change. Everyone else runs on paid capacity. Pay-as-you-go bills $0.01 per credit with no commitment. A capacity pack is $200 per month for 25,000 credits, which works out to $0.008 per credit, cheaper per credit than metering. The catch is that pack credits reset every month with no carryover, so the pack only beats pay-as-you-go if you actually consume most of it. Underused, a $200 pack you half-fill costs more per credit used than metering would have. Start metered, watch the real consumption, then buy a pack once usage is steady enough to fill one.

The automation identity has the dependency people forget twice. If a dedicated service account owns the flows, and the flows use premium connectors, that account needs a Power Automate Premium license. One Premium seat covers unlimited flows for that owner, which is far cheaper than a per-flow Process license the moment there is more than one flow. The forgotten half: the Office 365 Outlook connector sends as the connection owner, so if any flow sends email as the service account, that account also needs a mailbox-bearing Microsoft 365 license, or it cannot send at all. The service-account move is a Premium seat plus a mailbox, never a Premium seat alone.

One governance choice sits on top of all of it. Turning on Git source control turns the environment into a Managed Environment, and every user in a Managed Environment must hold a qualifying premium Power Platform license. If your app users and your service account are already premium-licensed, the Managed Environment adds governance without adding licenses, on one condition: no unlicensed user is ever added to that environment. That proviso is the thing to watch, because the day someone adds one, the cost of the control changes and no invoice announces it.

Here is the shape of bill one, with the dependency each line hides.


| Who or what                    | What it needs                                        | The dependency people miss                                            |
| ------------------------------ | ---------------------------------------------------- | --------------------------------------------------------------------- |
| App user                       | Premium Power Apps seat per user                     | At pilot volume, the seats are the largest recurring line             |
| Agent access by security group | Entra ID P1                                          | Group-based assignment is a paid Entra feature, not free              |
| Agent usage                    | Copilot Credits, or coverage by an M365 Copilot seat | An M365 Copilot seat zero-rates internal use; without it, you meter   |
| Flow owner (service account)   | Power Automate Premium                               | One Premium seat beats per-flow Process licensing past the first flow |
| Flow that sends email          | Mailbox-bearing M365 license on the sender           | The Outlook connector sends as the owner; no mailbox, no email        |
| Managed Environment            | Premium license for every user in it                 | One unlicensed user added later breaks the assumption                 |


## Bill two: consumption, the line that grows

The Azure consumption bill is modest at a pilot, and it does not stay that way. It has a fixed part and a variable part, and only one of them matters at scale.

The fixed part is the always-on floor: a search tier that bills the same whether the agent answers ten questions or ten thousand, plus storage that rounds to nothing. Keep the optional extras on their free allowances by deliberate choice rather than by accident. A semantic ranker on its free plan, for example, adds nothing to the tier until query volume crosses the allowance, at which point it starts billing per thousand queries.

The variable part is the model, and it is the line that grows fastest. Two structural facts decide it. Output tokens cost several times what input tokens cost, across essentially every current model, so a workload that generates a lot of text per request prices very differently from one that mostly reads. And the cost is per call, so a design that issues many model calls per task multiplies everything. When one task fans out into many reasoning calls, the call pattern, not the unit price, is the dominant lever.

Copilot Studio makes the same point in its own currency, and the rates are worth knowing because they stack. A generative answer is 2 credits. An agent action is 5. Tenant graph grounding is 10 credits per message, so a single graph-grounded generative answer is 12. The expensive multiplier is reasoning: when an agent uses a reasoning model, Copilot Studio bills the feature rate plus a premium AI-tools meter at 10 credits per 1,000 tokens on top. Reasoning is not a setting you leave on for comfort. It is a meter.

Bring your own model from Azure or Microsoft Foundry and the credit charge for the AI tool drops; you pay Foundry token costs instead, plus the agent action that invoked the tool. That moves the variable cost from the credit pool to the Azure bill, which is exactly why you cannot reason about one bill without the other.

A word on the Azure figures: this is the structure, not a quote. List prices for models and search tiers change, provisioned throughput rewrites the math again for sustained load, and the platform adds support, networking, and logging on top of the raw token meter. Pull the current numbers from the Azure pricing calculator at design time and reconcile them against your first real month. The durable claim is the shape, not the dollar amount: a fixed floor, a variable model cost, output dearer than input, and the call pattern as the lever.

## The shape, and what to do about it

Put the two bills side by side over a pilot's life and the shape holds. At low volume, the licensing and seats dominate and the Azure compute is a rounding error. As volume climbs, the model-token cost overtakes everything while the search tier stays a flat floor either way. The cost does not just grow. It changes which line you are managing.

That is the reframe worth carrying into a budget conversation. The line that will hurt you at scale is the one that is nearly invisible at a pilot. So govern it before it grows, not after.

- Forecast the model-call pattern before you build, not the headline token price. Microsoft publishes an agent usage estimator; use it, then distrust it until a real month reconciles it.
- Instrument per-task consumption early. You want credits-per-task and tokens-per-task from real usage before you choose a billing path, not after a pack runs dry.
- Set the caps that already exist. Copilot Studio supports per-agent monthly consumption caps and environment-level credit allocation, so one chatty agent cannot drain the tenant pool, and an environment with its own allocation is insulated from tenant-level overage.
- Right-size the service-account move to one Premium seat plus a mailbox, not a stack of Process licenses.
- Reconcile every forecast against the next invoice and a per-resource cost export, and revisit the model-call pattern first when the variable line climbs.

None of this is exotic FinOps. It is the ordinary discipline of knowing which meter you are on, applied to a system that happens to run on two at once. I made a version of this point about Code Apps in [the post on calling an LLM from a Code App](https://jakubbjl.github.io/2026/06/11/calling-llm-from-power-apps-code-apps.html), and again about agents in [the post on when the agent is the wrong answer](https://jakubbjl.github.io/2026/06/13/when-the-agent-is-the-wrong-answer.html): pick a pattern and you have picked a meter. The cost model is where that choice finally shows up as a number. That agent-versus-flow post is the companion to this one on the enforcement side, the tenant-wide disable conditions an agent inherits when the credit pool runs dry; this post is about what the meters cost, that one is about which outages they expose you to.

The licensing bill intersects governance, not only budget. The Managed Environment line above is the same control I classified in [the Power Platform security baseline](https://jakubbjl.github.io/2026/06/11/power-platform-security-baseline-default-enable-configure.html): a control you cannot license is not a control, it is a plan. Running cost and governance posture turn out to be the same conversation read from two ends.

Buy the seats and the floor with your eyes open. Govern the model-token meter before the agent scales. That is the line that grows.

## Caveats and limits

- Every price here is list price and ages. Copilot Credit rates, capacity-pack terms, Azure model and search prices, and Entra and Power Platform license prices all change; verify each at design and purchase time rather than from this post's date.
- The cost shape is a model, not a measurement. It describes one common pattern, a pilot dominated by seats moving to a production cost dominated by model tokens, and your workload may differ. Reconcile against real invoices.
- This post describes a generic Copilot Studio and Azure pattern, not any specific deployment, and I do not speak for Microsoft.
- Copilot Credit rates were verified against Microsoft Learn on June 17, 2026. The Azure model pricing structure was checked the same day against current published rates; treat the specific Azure figures as illustrative and pull live numbers at design time.

## References

- [Copilot Credits overview](https://learn.microsoft.com/microsoft-copilot-studio/copilot-credits-overview) (pay-as-you-go at $0.01 per credit; capacity pack at $200 for 25,000 credits per month; pre-purchase plan)
- [Billing rates and management](https://learn.microsoft.com/microsoft-copilot-studio/requirements-messages-management) (per-feature credit rates; reasoning models stack a premium AI-tools meter; bring-your-own-model billing)
- [Copilot Studio licensing](https://learn.microsoft.com/microsoft-copilot-studio/billing-licensing) (Copilot Credits as the common currency; the September 2025 rename from messages; the Microsoft 365 Copilot metering exemption)
- [Use Copilot Studio prepaid capacity packs](https://learn.microsoft.com/microsoft-365/copilot/pay-as-you-go/copilot-capacity-packs) (capacity packs reset monthly with no carryover; pay-as-you-go overage; credit policies)
- [Forecast agent consumption with the agent usage estimator](https://microsoft.github.io/copilot-studio-estimator/) (model credit consumption before you build)
- [Optimize Copilot Credit costs with a pre-purchase plan](https://learn.microsoft.com/azure/cost-management-billing/reservations/copilot-credit-p3) (tiered discounts; the only option that discounts the credit)