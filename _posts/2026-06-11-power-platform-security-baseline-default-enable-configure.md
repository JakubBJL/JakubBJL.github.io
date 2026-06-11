---
layout: post
title: "The controls you assumed were on: a DEFAULT / ENABLE / CONFIGURE baseline for Power Platform"
date: 2026-06-11T00:00:00.000Z
tags:
  - power-platform
  - governance
  - security
  - architecture
human-approved: true
---

Most Power Platform security assessments I read spend their first ten pages re-describing the platform. Identity is Entra ID, data is encrypted, environments are isolated. All true, all enforced by Microsoft, and all copied from the last assessment, which copied them from the one before. Meanwhile the findings that actually hurt, the unaudited table, the environment open to every licensed user in the tenant, the flow run history displaying credentials in plain text, come from somewhere else entirely. They come from controls everyone in the room assumed were on.

The fix I have settled on is not a longer checklist. It is a classification. Sort every platform control by who does the work: the platform, a setting, or a design decision. Write that down once as a baseline, and let every project assessment document only its deviations. The categories are DEFAULT, ENABLE, and CONFIGURE, and the security gaps live almost entirely in the seam between the first and the second.

## DEFAULT: what the platform enforces by design

Some controls are active in every deployment with no project action. Identity runs exclusively through Microsoft Entra ID. Data is encrypted at rest and in transit. Environments are isolated containers, and an app in one environment cannot reach connections or data in another. Credentials for data sources live in Connection objects rather than in code or configuration.

The work these controls demand is verification, not implementation: confirm nobody bypassed them, for example by embedding a secret in an environment variable instead of a connection. That check takes minutes. The mistake is spending assessment effort here while the next category goes unread.

## ENABLE: the dangerous middle

ENABLE controls exist in the product, cost nothing extra in most cases, and are off until someone turns them on. This is where assumed-on meets actually-off.


| Control                   | What teams assume                                 | What the platform does until someone acts                                                                                                                                 |
| ------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Dataverse change auditing | "Auditing is on, it is an enterprise platform"    | Auditing must be turned on at the environment level, and then per table. An environment with auditing on still logs nothing for a table whose own audit setting is off.   |
| Read access logging       | "If writes are logged, reads are logged"          | Read auditing is a separate setting from change auditing, off by default, and applies to production environments with the required Microsoft 365 licensing.               |
| DLP                       | "The tenant has connector policies"               | No data policies exist in a tenant by default. A policy also does nothing for an environment it is not scoped to.                                                         |
| Flow run history          | "Run logs are operational metadata"               | Run history shows the inputs and outputs of every action in plain text. Hiding sensitive values requires Secure inputs and Secure outputs on each individual action.      |
| Environment access        | "Only the project team can reach our environment" | An environment with no security group assigned is open to every user in the tenant who holds a Dataverse license.                                                         |
| Cross-tenant connections  | "Our data stays in our tenant"                    | With tenant isolation off, which is the default, a user with valid credentials can connect Power Platform resources to another organization's tenant in either direction. |


Two of these deserve a sentence more.

The auditing rows are the ones I find most often in the field. Teams check the environment-level switch, see it on, and close the line item. The per-table setting, "Audit changes to its data," sits in each table's advanced options and is its own decision, and read logging is a third, separately licensed decision on top. Three switches, one assumption.

The security group row carries a trap in the other direction. The control itself is simple: every production environment gets an Entra security group. But assign a group to an environment that already has users, and everyone not in the group is disabled immediately. Populate first, assign second. The remediation for an open environment can cause the outage.

What makes this category dangerous is precisely that nothing in it is exotic. Every row is a documented setting with a documented default. The failure is organizational: ENABLE items look like DEFAULT items from a distance, and nobody owns the act of flipping them.

## CONFIGURE: what only design can do

The third category requires decisions, not switches.

Least privilege is the obvious resident. Dataverse's out-of-box roles grant broad access, and minimal roles for an actual workload have to be designed and assigned. Privileged access is similar work at the admin tier: without Privileged Identity Management, a system administrator holds the role around the clock, and without access reviews, nobody removes the administrator who changed teams a year ago.

One CONFIGURE decision is irreversible and belongs in every design review: Dataverse table ownership. Whether a table is user-owned or organization-owned is chosen at creation and cannot be changed afterward, and organization-owned tables reduce security to a binary, a user either holds the privilege on all rows or on none. Choose wrong on day one and row-level security for that table is gone for the table's lifetime. The schema decision is a security decision.

The rest of the category is where posture meets budget, because several CONFIGURE controls are gated by licensing. Data masking on secured columns and the IP firewall both require Managed Environments. Conditional Access, the mechanism that makes MFA enforceable, is an Entra ID P1 feature. Key Vault integration and virtual network support run on Azure consumption. Tenant-level hygiene rounds it out: environment creation is open to users with eligible licenses until it is restricted to specific admins in tenant settings, and an ungoverned environment bypasses every environment-scoped control on this page.

A control-to-license map belongs in the baseline for exactly this reason. It converts a security requirement into a line item a project can plan for, instead of a surprise that arrives mid-implementation. Many baseline controls need nothing beyond standard entitlements. The ones that do should never be discovered late.

## The payoff: assess deviations, not the platform

Here is what the classification buys you. The baseline is written once, owned centrally, and referenced by every project. A project security assessment then records four things: exceptions, where a baseline control is not active and why; deviations, where it is implemented differently; additions, where the workload needs controls beyond the baseline; and risks specific to the solution. Each entry uses the same three categories, so a reviewer knows immediately whether a gap is a missing platform guarantee, an unflipped setting, or a design shortfall.

Assessments shrink from descriptive documents to decision records. Reviews get faster because the reviewer reads ten pages of deltas instead of fifty pages of platform description. And the dangerous middle stops being dangerous, because the ENABLE list exists as an explicit, owned artifact instead of a shared assumption.

I wrote earlier about [account discipline in Power Platform](https://jakubbjl.github.io/2026/06/10/power-platform-named-accounts-service-principals.html), and the argument ends the same way this one does: governance artifacts like these are what make speed possible, because every project that does not re-litigate the platform is a project that ships sooner. The baseline is the cheapest piece of security work an organization can commission. It is written once, and it is mostly a catalog of switches.

The platform is not insecure by default. It is unconfigured by default. Those are different problems, and only one of them is yours.

## References

- [Manage Dataverse auditing](https://learn.microsoft.com/power-platform/admin/manage-dataverse-auditing), Microsoft Learn
- [Control user access to environments with security groups and licenses](https://learn.microsoft.com/power-platform/admin/control-user-access), Microsoft Learn
- [Implement a data policy strategy](https://learn.microsoft.com/power-platform/guidance/adoption/dlp-strategy), Microsoft Learn
- [Manage sensitive input like passwords in Power Automate](https://learn.microsoft.com/power-automate/how-tos-use-sensitive-input), Microsoft Learn
- [Cross-tenant inbound and outbound restrictions](https://learn.microsoft.com/power-platform/admin/cross-tenant-restrictions), Microsoft Learn
- [Control who can create and manage environments](https://learn.microsoft.com/power-platform/admin/control-environment-creation), Microsoft Learn
- [Security concepts in Microsoft Dataverse: table and record ownership](https://learn.microsoft.com/power-platform/admin/wp-security-cds), Microsoft Learn
- [Create and manage masking rules](https://learn.microsoft.com/power-platform/admin/create-manage-masking-rules), Microsoft Learn
- [Identity and access management in Power Platform](https://learn.microsoft.com/power-platform/admin/security/identity-access-management), Microsoft Learn