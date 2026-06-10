---
layout: post
title: "A service account is an owner, not a developer"
date: 2026-06-10
tags: [power-platform, governance, security, power-automate]
human-approved: true
---

There is a pattern I keep meeting in Power Platform estates that have grown faster than their governance: a small pool of shared service accounts, passed among developers, used for everything. Building apps, authoring flows, creating connections, importing solutions. The team will usually defend it as best practice. It feels prudent, because the flows survive when people leave, and it feels efficient, because nobody has to manage per-developer access.

It is neither. The practice collapses two concerns that Microsoft's own guidance treats as separate: who authors a thing, and who owns it at runtime. Keep those apart and the account model designs itself. Merge them and you inherit three control failures at once.

## The principle: authoring and ownership are different concerns

The reason a service account exists is runtime ownership. A production flow owned by a personal account breaks when that person leaves, loses a license, or changes roles, so Microsoft recommends decoupling mission-critical flows from any individual's lifecycle. That is an ownership problem, and ownership is what non-human identities are for.

Authoring is a different problem. Building flows, apps, and solutions is interactive human work, and everything about the platform's control surface, from security roles to audit logs to Center of Excellence telemetry, assumes it can attribute that work to a person.

A shared development account answers the ownership question with a tool meant for it, then quietly breaks the authoring question with the same tool. The fix costs nothing structural: developers build under their own named Microsoft Entra accounts, dedicated non-human identities own production flows and connections, and environment administration runs under named admin accounts. Three jobs, three credential types.

## Failure one: the blast radius

Least privilege applies to service accounts themselves. An identity whose job is to own flows and connections needs exactly the security roles that let those flows run. It has no legitimate need for System Customizer, System Administrator, or any broad maker role. Dataverse's predefined security roles are explicitly built around minimum required access, which makes over-provisioning a choice, not a default.

A shared development account inverts this. To be useful for building, it must hold broad maker privileges; to own production, it holds runtime privileges; shared among several people, it concentrates both into one credential whose password, by construction, several humans know. Any compromise or misconfiguration now has the widest possible blast radius the team could have designed. Separate credentials for separate purposes is the entire control.

## Failure two: attribution

This is the strongest argument, because it is the one you cannot compensate for later. Power Platform logs maker activity, app and flow creation, modification, connection changes, against the account that performed the action, surfaced through Microsoft Purview audit. Governance reporting, incident investigation, and compliance reviews all hang off that attribution.

Microsoft's licensing documentation states the problem with unusual directness: user accounts shared as service accounts pose a security risk, and it is difficult to track who made changes to a flow when multiple people have access to the account. Shared credentials also remove the ability to revoke or investigate at the level of an individual. When something goes wrong in production, "it was the service account" is the complete and final answer your audit log will give you.

The same failure hollows out CoE tooling. The CoE Starter Kit treats maker identity as a core signal for adoption, risk, and ownership dashboards. Feed it shared accounts and the governance reporting your organization relies on describes a fictional maker who never goes on leave and never leaves the company.

## Failure three: the licensing trap

Here is the part most teams have not read. Microsoft's Power Automate licensing FAQ, in its multiplexing section, says plainly that service accounts are not recommended as a best practice, and that where the goal is removing the dependency on a flow's original owner, the answer is a service principal. Shared service accounts running premium flows also walk straight into multiplexing rules, where the users behind the shared credential may each need licensing of their own.

The current recommended owner for mission-critical flows is a service principal application user: a non-human Entra identity that cannot log in interactively, cannot have its password shared around a team, holds exactly the roles you grant it, and carries its own documented licensing model (a Power Automate Process license for premium flows, with defined request limits). The platform's guidance for ALM points the same direction, with service principals owning flows and connections deployed through pipelines. The era in which a shared mailbox-style account was the only way to keep a flow alive ended some time ago. The guidance moved. Many estates did not.

## The mapping

| Activity | Identity |
|---|---|
| Building apps, flows, solutions | The developer's own named Entra account |
| Owning production flows and connections | Service principal application user (preferred) or a dedicated, non-interactive service account |
| Administering environments | Named admin account, or a dedicated admin principal |

Operating rules that keep the model honest: provision flow-owner identities with only the roles their flows require, review those grants on a cycle, never use an owner identity for interactive development, and treat any maker action that cannot be traced to a named person as a finding, not a convenience.

## Caveats

Service principals have edges to plan for: they cannot co-own flows, non-solution flows need connections explicitly shared with the application user, and premium flows under principal ownership need a Process license. None of these costs outweighs attribution. And if your estate runs on shared accounts today, the migration is unglamorous but mechanical: inventory what each shared account owns, move ownership to principals, then revoke the interactive credentials.

I have written before that governance is what enables speed rather than what brakes it. Account discipline is the cheapest example of that claim in the entire platform. It requires no new licenses for the developers, no new tooling, and no process redesign. It only requires deciding that knowing who did what is a property your platform must have.

A service account is an owner. The moment it becomes a developer, you have traded your audit trail for a shared password.

## References

- [Understand flow ownership and access](https://learn.microsoft.com/power-automate/guidance/coding-guidelines/understand-access-to-flows), Microsoft Learn
- [Support for service principal owned flows](https://learn.microsoft.com/power-automate/service-principal-support), Microsoft Learn
- [Power Automate licensing FAQ: multiplexing](https://learn.microsoft.com/power-platform/admin/power-automate-licensing/faqs), Microsoft Learn
- [Role-based security roles for Dataverse](https://learn.microsoft.com/power-platform/admin/database-security), Microsoft Learn
- [View Power Apps activity logs in Microsoft Purview](https://learn.microsoft.com/power-platform/admin/activity-logging-auditing/activity-logs-power-apps), Microsoft Learn
