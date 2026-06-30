---
layout: post
title: "The seams, not the steps: assembling a multi-platform Copilot Studio and Azure agent"
date: 2026-06-29
tags: [copilot-studio, azure, architecture, agents]
human-approved: true
---

A serious agent rarely lives on one platform. The one I have in mind takes in a document, runs it through an Azure backend of functions and search and storage, writes structured results into Dataverse, and surfaces them through a Copilot Studio agent, with intake handled by a form and a Power Automate flow. Seven platforms, one agent. The way these get built, almost always, is as a set of standalone scripts, one per platform. Each script is clean. Each was written by someone who knew that platform. Open any one of them and it reads like it should work.

The build still stalls. Not inside the scripts, which are fine, but between them. A function script assumes a Dataverse identity that no Dataverse script created. A topic script assumes an intake form that lives in a platform the topic author never touched. The Azure script stands up storage the Power Platform side expects to already exist. Every script silently depends on something that lives somewhere else and does not exist yet, and no script owns the dependency, because it belongs to neither platform. The seams between the scripts, not the steps inside them, are where a multi-platform agent build stalls. That is the part worth designing for, and it is the part the scripts hide.

## Make the dependencies something you can see

The first move is to stop treating the build as a stack of scripts and start treating it as a graph of dependencies. Lay it out as a matrix: one row per build step, and for each step, what it depends on, which platform it touches, and which script, if any, actually covers it.


| Build step                           | Depends on                                        | Platform                   | Covered by  |
| ------------------------------------ | ------------------------------------------------- | -------------------------- | ----------- |
| Function writes results to Dataverse | App registration, application user, security role | Azure, Entra, Dataverse    | not covered |
| Agent gated to reviewers only        | Entra security group, enterprise app assignment   | Entra, Copilot Studio      | not covered |
| Topic shows real results             | Tool flow wired, backend returning data           | Copilot Studio, Azure      | partial     |
| Flow reads the source document       | Intake form and library verified to exist         | SharePoint, Power Automate | assumed     |


The cells that read "not covered" are the actual work. They are the seams. No single platform script owns them precisely because they span platforms, and so in a script-by-script plan they are the steps that are nobody's job until the build stops moving and someone has to find out why. The matrix is not documentation you write afterward. It is the plan, and the empty cells are the schedule.

## The three seams that recur

Across this class of build, the same three seams show up almost every time.

The identity seam, between Azure and Dataverse. An Azure Function authenticates its own callers with a function key. That key says nothing to Dataverse. To write a row, the function needs an identity Dataverse recognizes: an Entra app registration, registered inside Dataverse as an application user, bound to a custom security role scoped to exactly the tables it touches. The application user is unlicensed and authenticates as a service principal rather than as a person, which is the documented server-to-server pattern. Scope the role tightly: Create, Read, Write, Append, and Append To on the result tables, Delete denied, read-only on reference data. This work belongs to no single platform script. It has to be scheduled as its own step, and it has to happen before any function that depends on it can be tested.

The storage seam, between SharePoint and the function. The function needs the document's content, and the direct path is to give it Graph or SharePoint application permissions and let it read the library itself. Resist that path. Every application permission you grant the function is a permission you then have to scope, govern, and defend for the life of the system. The narrower pattern is to let Power Automate, which already holds a governed SharePoint connection, read the file and hand the content to the function over HTTP or by dropping it into blob storage the function already owns. The broker keeps the function's identity narrow and removes a whole class of permissions from the build. Microsoft's own guidance for securing functions is to run them with the lowest possible permissions, and the broker is how you honor that at the seam rather than at the code.

The access seam, at the published agent. Publish without thinking about access and the agent answers whoever can reach the channel. Gate it. Put the published agent behind an Entra security group: set the enterprise application to require user assignment, then assign only the reviewer group. This is not free. Group-based assignment to an enterprise application requires Microsoft Entra ID P1, so the license is a design input, not a detail to discover at the end. Open access during a test phase is defensible, but as a deliberate toggle, not a default someone forgot to close.

## Some steps cannot be finished in order, and that is fine

Even with the dependencies mapped, parts of the build cannot complete in a single forward pass, because a step depends on work from a later step. The honest answer is not to force a linear order. It is to plan the circle-back.

Build the Copilot Studio topics first with placeholders, knowing the data is not there yet. Stand up the backend: the functions, the data tables, the tool flows. Then return to the topics and wire the tool flows in, replacing each placeholder with a real call. A topic that showed placeholder text now gets a call-an-action node feeding a variable; a shared run identifier held in a global variable lets the topics correlate to the same case; the placeholder becomes a reference to live data. Publishing the agent is the last circle-back of all, after every wiring step, never before.

This is why a dependency-ordered sequence beats a platform-ordered one. Worked in dependency order, the build goes: the agent shell with placeholders, then the service identity, then the cloud resources, then the admin and security objects, then the data tables, then verification that the existing storage and intake are really there, then the code deployment, then the flows, then the topic wiring, then the final circle-backs and publish, then one end-to-end test. Platform order would group all the Azure work, then all the Power Platform work, and it would strand the identity step and the topic wiring exactly where they hurt most.

Two steps in that list earn their place specifically. Verify the pre-existing pieces rather than assuming them: the intake form, the storage layout, the trigger flow are the easiest things to treat as already present and the most common place a build quietly breaks. And treat the end-to-end test as the only step that depends on everything. Submit one real document and confirm the whole chain: the flow triggers, extraction runs, the content indexes, the backend processes it, the results land, and the agent surfaces them on request. Any earlier step passing in isolation is necessary and not sufficient.

## The same discipline is what moves it

The build is not the last time you meet these seams. The first time the finished agent moves out of the tenant it was built in and into a client's, every one of them returns, along with a few that only a tenant move surfaces.

Only the Power Platform side packages into a solution: the Dataverse tables, the flows that live inside the solution, the agent and its topics. The Azure resources and the Entra registrations do not travel in a solution file and are rebuilt in the target. No secret travels either, by design, because keys and connection passwords are never inside an export. You recreate every secret in the target, and anything you were carrying in plain configuration moves to Key Vault on the way. On import, the connection references re-point to identities signed into the new tenant, and the dependency-order rule that governed the build governs the move just as strictly: do not import the solution before the Azure endpoints its flows expect already exist, or those flows bind to nothing.

I have written separately about keeping a hybrid build like this in source control and promoting it as [two planes moving in opposite directions](https://jakubbjl.github.io/2026/06/15/hybrid-power-platform-azure-source-control-two-planes.html), and about serving many production targets from [one solution without forking it per market](https://jakubbjl.github.io/2026/06/25/multi-region-alm-power-platform-one-solution-many-productions.html). The tenant handover is the same discipline pointed at a different moment. Build, promote, hand over: each is a different verb over the same graph, and each rewards naming the seams before working the steps.

## Caveats

- The platform behavior here was verified on Microsoft Learn in June 2026. The server-to-server application-user pattern, the function security guidance, and the Entra group-assignment licensing are stable, but re-verify at design time rather than from this post's date.
- The broker pattern is a recommendation, not a platform requirement. A function can hold its own SharePoint permissions, and sometimes the simpler topology is the right call. The point is that how wide the function's identity should be is a governance decision, and it deserves to be made on purpose. Left alone, the easy path makes it for you.
- This is a design-time discipline. Mapping the seams before the first script is cheap. Discovering them one stalled build at a time is the expensive way to learn the same thing.

## References

- [Build web applications using server-to-server (S2S) authentication](https://learn.microsoft.com/power-apps/developer/data-platform/build-web-applications-server-server-s2s-authentication) (the unlicensed application user, the service principal identity, and the custom security role that scopes it)
- [Use OAuth authentication with Microsoft Dataverse: connect as an app](https://learn.microsoft.com/power-apps/developer/data-platform/authenticate-oauth) (create the custom security role first, then the application user bound to it)
- [Securing Azure Functions](https://learn.microsoft.com/azure/azure-functions/security-concepts) (run the function app with the lowest possible permissions; organize functions by privilege)
- [Secure your Copilot Studio projects](https://learn.microsoft.com/microsoft-copilot-studio/guidance/sec-gov-phase3) (manage access through Entra ID groups and group teams; gate release to production)
- [Manage access to an application](https://learn.microsoft.com/entra/identity/enterprise-apps/what-is-access-management) and [Assign users and groups to an application](https://learn.microsoft.com/entra/identity/enterprise-apps/assign-user-or-group-access-portal) (require user assignment; group-based assignment requires Microsoft Entra ID P1 or P2)
- [Environment variables for Power Platform](https://learn.microsoft.com/power-apps/maker/data-platform/environmentvariables) (externalize references for ALM; remove a value before export so a new one is supplied on import; use Key Vault for secret-typed values)

A multi-platform agent is not built by running each platform's script in turn. It is assembled across the seams between them, in the order the dependencies allow, and the work that no single script owns is the work that decides whether it ships. Name the seams first. The scripts were never the hard part.