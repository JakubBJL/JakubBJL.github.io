---
layout: post
title: "When the agent is the wrong answer: deterministic notification belongs in a flow"
date: 2026-06-13T00:00:00.000Z
tags:
  - power-platform
  - agentic-ai
  - governance
  - architecture
human-approved: true
---

This request comes up often now: build an agent that watches our records and tells the right people when something needs attention. Reminders before a due date. An alert when an item goes overdue. A nudge when a step is skipped. The word "agent" is in the request, so the design starts there.

Most of that work does not need an agent. It needs a flow. The reflex to reach for an agent is the expensive way to build a notifier, and the reason is not taste. It is what each tool is for, which meter each one moves, and which failure mode each one inherits.

## The line Microsoft draws, in its own words

Copilot Studio's documentation is unusually clear about the distinction, and it is the distinction the whole decision rests on. Topics and agent flows are described as "deterministic pathways for automation. Given the same inputs, they'll always produce the same output, making them reliable and predictable." Tools the orchestrator calls are expected to "behave deterministically given the same inputs, since the orchestrator treats them as reliable functions." The judgment lives in one place: the generative orchestrator, the LLM that turns an event or a message into a plan, chooses which tools to call, and sequences them.

So the architecture splits cleanly. Deterministic decisions over known fields are flow territory. Judgment over ambiguous input is orchestrator territory. An autonomous agent is real and supported, with event and scheduled triggers that fire without a user present, generally available since March 2025. The question is never whether you *can* build the notifier as an autonomous agent. You can. The question is whether the decision the notifier makes needs the orchestrator's judgment at all.

For a reminder before a due date, it does not. The decision is `due_date - today <= threshold`. For an overdue alert, the decision is `status = open AND due_date < today`. For a skipped-step nudge, the decision is a missing value in a column. These are rules over fields. They are exactly the "same input, same output" case the documentation reserves for flows. A Dataverse trigger with a filter expression, a condition, and a Teams message covers the entire requirement, and it covers it more predictably than an LLM would.

## The test

Before you build, ask one question of the decision the automation makes, not of the workload around it.

Does the decision require judgment over input you cannot fully specify in advance? If yes, you are in agent territory. If the decision is a rule you can write down as a condition over known columns, it is a flow, and wrapping it in an agent buys you nothing the flow lacks.

This is a narrower test than "is the process complex." A five-step lifecycle with branching, escalations, and role-based routing can be entirely deterministic. Complexity is not the signal. Ambiguity is. The orchestrator earns its place when the system has to interpret something, not when it has to follow a rule that happens to have many clauses.

## Follow the money

The distinction is not academic, because the two paths bill differently and fail differently.

A cloud flow in Power Automate triggered by a Dataverse row change runs against Power Automate licensing. It does not touch Copilot Studio capacity. An autonomous Copilot Studio agent doing the same job consumes Copilot Credits per run. Generative orchestration is billed as an autonomous action plus the actions it invokes. At the published rates, an agent action is 5 Copilot Credits, a generative answer is 2, and tenant graph grounding is 10. On pay-as-you-go, a Copilot Studio message is metered at a cent each. None of that buys a better reminder. It buys judgment the reminder never uses.

Then there is the coupling, which is the part teams discover later. Copilot Studio's overage enforcement disables custom agents tenant-wide at 125 percent of prepaid capacity. Agent flow enforcement blocks new agent-flow runs once prepaid capacity is fully consumed. Both are tenant-level conditions. A deterministic notifier built as an agent now shares a disable switch with every other agent in the tenant, and the thing that trips that switch may be a workload it has nothing to do with. Your overdue alerts can go dark because a different team's chatbot had a busy month. A Power Automate flow carries no such shared fate.

Pick a tool and you have picked a meter, and you have picked which outages you are exposed to. I made the same point about Code Apps in [the post on calling an LLM from a Code App](https://jakubbjl.github.io/2026/06/11/calling-llm-from-power-apps-code-apps.html): every pattern is also a billing decision and a governance decision. Notification logic is the cleanest case. The cheaper, more isolated, more predictable tool is also the correct one.

## When the agent is the right answer

This is not an argument against agents. It is an argument for spending them where they earn it. Two conditions justify the agent layer even when much of the surface work looks like notification, and I have seen both hold in a real design for a regulated organization.

The first is genuine judgment over unstructured input. One part of that system checked uploaded documents for the presence of required elements, reading text the rules could not anticipate and classifying each required element as present, missing, or unclear. That is not a rule over a column. It is interpretation, and it belongs to the orchestrator and a model. The design lesson worth carrying is the restraint around it: the feature was deliberately scoped to report presence and absence, not to score quality or assign a rating, and every output carried a fixed statement of what it did and did not cover. An AI feature that survives governance review is usually one that claims less than it could.

The second is the control plane, not the task. When every automated action has to be provable after the fact, when a single place must hold the connectors and enforce the runtime limits, and when the components that act must carry no credentials of their own, the agent layer is buying you a hardened, auditable, centrally governed surface. In that design the orchestrator held the connections and the worker components communicated only through a request-and-result protocol, holding no credentials. Append-only logging recorded what was authorized, what happened, and what the system was prevented from doing. None of that is required to send a Teams message. All of it is required to send a Teams message in an environment where you must later prove who authorized the action and that nothing else was possible. The agent is justified by the governance requirement, not by the notification.

That is the honest read on the build: most of the notifiers could have been flows, and saying so out loud is what makes the case for the agent layer credible where it actually applies. A design that needs an agent everywhere usually needs one nowhere.

## A decision checklist

Run a proposed "notification agent" through these before building. If every answer points to a flow, build a flow.

- Is the trigger a record event or a schedule? Both are native Dataverse and Power Automate triggers. Neither requires an agent.
- Is the decision a condition over known columns? If you can write it as a filter expression, it is deterministic. Flow.
- Does any step interpret input you cannot specify in advance? If no, there is no work for the orchestrator. Flow.
- Will the action ever need to be defended in an audit, with proof of authorization and proof of what was prevented? If yes, the agent's auditable, centrally constrained action surface may justify it, independent of the notification.
- Do the acting components need to hold no credentials of their own? If centralized credential isolation is a requirement, that is a control-plane reason for an agent, not a notification reason.
- Which meter do you want to move, and which tenant-wide disable conditions are you willing to inherit?

The first three questions resolve most "notification agent" requests to a flow. The last three are the only ones that pull the decision back toward an agent, and they are about governance and reliability, never about the notification itself.

## Caveats and limits

- Autonomous triggers are not available in some sovereign and government clouds. Confirm trigger availability for your tenant before committing to an agent design that depends on them.
- This is a design-time decision. Once a notifier is live as an agent, moving it to a flow is a rebuild. The cost of getting it wrong is paid later, which is exactly why the question belongs at the front.
- Copilot Credit rates, enforcement thresholds, and the meter that applies to each event change over time. Verify the current rates and the current enforcement behavior at design time rather than from this post's date.
- A flow is not free of governance. It still has an owner, a connection, and a run-as identity, and those are decisions in their own right. The point is narrower: the flow does not add a judgment layer or a metered, tenant-coupled capacity surface that the work does not use.

## References

- [Agent flows in Microsoft Copilot Studio FAQ](https://learn.microsoft.com/microsoft-copilot-studio/flows-faqs) (topics and agent flows as deterministic pathways; agent flows versus Power Automate cloud flows)
- [Design autonomous agent capabilities](https://learn.microsoft.com/microsoft-copilot-studio/guidance/autonomous-agents) (event-driven autonomy, scoped permissions, auditable processes)
- [Apply generative orchestration capabilities](https://learn.microsoft.com/microsoft-copilot-studio/guidance/generative-orchestration) (the orchestrator as the judgment layer; event triggers for autonomy)
- [Trigger flows when a row is added, modified, or deleted](https://learn.microsoft.com/power-automate/dataverse/create-update-delete-trigger) (Dataverse event triggers, filter expressions, run-as)
- [Billing rates and management](https://learn.microsoft.com/microsoft-copilot-studio/requirements-messages-management) (Copilot Credit rates; overage enforcement at 125 percent; agent flow enforcement)

The word in the request is "agent." The right first move is to ignore it and look at the decision. If the decision is a rule, you are building a flow, and the agent was never the answer.