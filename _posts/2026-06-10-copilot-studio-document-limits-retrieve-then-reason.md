---
layout: post
title: "Where Copilot Studio stops reading documents, and the ladder out"
date: 2026-06-10
tags: [copilot-studio, architecture, agentic-ai, azure-ai-search, rag, power-platform]
human-approved: true
---

Copilot Studio agents demo well on documents. You attach a SharePoint library, ask a question, and a cited answer comes back in seconds. Then the same agent meets production content: a 300-page technical specification, a folder of contracts, a compliance check that has to hold up against a defined set of rules. The answers get vague, the citations land in the wrong section, and the team concludes the data must be bad.

The data is usually fine. The agent has crossed a boundary the product does not announce: Copilot Studio's built-in knowledge is a conversational retrieval layer, not a document-intelligence engine. It behaves like a neighborhood library. It will find you a relevant page quickly. It will not read three books cover to cover and reconcile them.

This post maps that boundary twice, once from the documentation and once from the field, and then lays out the escalation ladder I use when a build crosses it. I have argued on LinkedIn that agents have an architecture problem, not a data problem. This is the platform evidence behind that argument.

## The boundary the documentation draws

The limits are not properties of the product. They are properties of the ingestion path. The same PDF faces a different ceiling depending on how it enters the agent.

| Ingestion path | File size ceiling | Conditions |
|---|---|---|
| SharePoint as knowledge source, no Microsoft 365 Copilot license | 7 MB | Enhanced search results must be off |
| SharePoint as knowledge source, with M365 Copilot license in the agent's tenant | 200 MB | Enhanced search results (Work IQ) must be on |
| Uploaded files (file upload knowledge) | 512 MB | Per file; .xls and .xlsx cap far lower in some agent types |
| Unstructured data sources (SharePoint or OneDrive upload path) | 512 MB | Up to 1,000 files, 50 folders, 10 subfolder levels per source |

A few more documented numbers worth designing around: an agent takes up to 500 uploaded files (the cap does not apply to SharePoint sources), generative answers consume at most 15 search snippets per response (a limit that applies across all knowledge topics combined), unstructured sources re-synchronize on a four-to-six-hour cycle rather than in real time, and files carrying confidential or highly confidential sensitivity labels, or password protection, are indexed with a Ready status but silently produce no answers. That last one is documented, and it is still the single most confusing failure mode I see in the field, because the knowledge panel reports success.

Look at the 7 MB versus 200 MB versus 512 MB spread. The constraint follows the retrieval infrastructure behind each path, not the file. Anyone sizing a document workload against "the Copilot Studio limit" is asking an underspecified question. The right question is: which ingestion path, under which license, with which features on.

## The boundary the documentation does not draw

A second set of limits never appears in a quotas table, because they are behavioral. I am separating them deliberately from the documented facts above: these come from repeated community measurement and from my own build experience, and you should treat them as field observations to verify against your own content, not as product specifications.

Answer quality degrades well before the size ceilings. Community testing has repeatedly placed reliable question answering somewhere below roughly 7,500 words per document, with reliable rewriting degrading earlier. Long documents also exhibit the familiar lost-in-the-middle effect: the model attends to the beginning and end and skims the interior, which is exactly where the clause you care about lives.

The built-in ingestion gives the maker no control surface. You cannot choose chunk size or overlap, cannot pick keyword versus vector versus hybrid retrieval, and cannot turn on semantic reranking from the built-in interface. When retrieval quality is the problem, there is no knob to turn. Long sessions can also overflow the model's context budget when large snippets accumulate, and the knowledge management UI itself becomes hard to operate past a low hundreds count of files.

One orchestration trap deserves its own paragraph because it costs teams days: with generative orchestration on, the orchestrator can intercept a request you intended for a deterministic topic and answer it from knowledge instead, so a carefully designed output template never runs. If a flow must be deterministic, disable generative actions for it or remove the knowledge source from that path. The platform's flexibility is the bug here, not the model.

## The anti-pattern that finds this boundary fastest

Every failed document build I have reviewed shares one shape: it tries to do document intelligence in real time, inside the conversation.

One build I worked on, a document-review agent in a regulated environment, had to check long technical documents against a defined set of quality rules and cross-reference relevant legislation. The first version sent the entire document to the model once per rule. The same content was tokenized and processed again for every check, runtimes stretched to many minutes per document, AI Builder credit consumption scaled with document length times rule count, and the platform's context limits silently truncated the longest source material. No retrieval layer, no structure extraction, documents treated as opaque blobs.

The lesson generalizes: every durable document workload I have seen pre-processes documents offline, before anyone asks a question. The conversation is where answers are delivered, not where reading happens.

## The ladder out

When a build crosses the boundary, there are three rungs, in order of cost.

### Tier 1: stay native, design around the limits

No new services, days of effort. Split documents below the quality threshold rather than the size ceiling. Manage knowledge files programmatically through Power Automate instead of hand-feeding the UI. Chain a large task into staged passes with a final aggregation step. Reset or summarize conversation context in long sessions. Turn off general knowledge so the agent answers only from supplied sources. These are real fixes for FAQ-shaped workloads that drifted slightly past their comfort zone. They do not fix retrieval quality, because nothing on this rung gives you control of retrieval.

### Tier 2: replace retrieval with your own

Copilot Studio now supports custom knowledge sources through the `OnKnowledgeRequested` trigger: a topic with this trigger becomes a knowledge source, calls your search endpoint with the platform's context-rewritten query (`System.SearchQuery` or `System.KeywordSearchQuery`), and writes formatted results into `System.SearchResults` for generative answers to consume. Pointed at Azure AI Search, this buys you hybrid retrieval, semantic reranking with a score threshold you choose, metadata filters, and field selection.

Two honest caveats. First, this is configurable only in code view, in YAML; there is no visual designer support, so it sits outside what most makers will discover on their own. For a long stretch this pattern lived purely in community write-ups; Microsoft has since published official guidance for it, which closes a documentation gap but not the tooling one. Second, it is a query-time solution only. You gain control of the query and ranking, and still own none of the chunking, embedding, parsing, or index schema. Those live wherever your index is built.

### Tier 3: retrieve-then-reason behind the agent

The enterprise pattern moves both retrieval and reasoning into a purpose-built pipeline and demotes Copilot Studio to what it is excellent at: the conversational front end. Document Intelligence extracts structure on ingestion (headings, tables, key-value pairs). Azure AI Search chunks and indexes for hybrid retrieval; starting points that have held up for me are chunks around 2,000 characters with 500 of overlap and a semantic reranker threshold around 2.5 on its 0-to-4 scale, tuned from there. Azure OpenAI does the reasoning with full prompt control. Functions or Logic Apps orchestrate offline ingestion and any multi-step checking workflow, batching rules into groups per model call and storing each verdict with quoted evidence and a confidence score, with low-confidence results routed to human review.

Retrieve-then-reason is the single highest-impact change on the ladder. It converts brute-force full-document scanning into targeted reasoning over retrieved chunks, which is the difference between a workload that cannot run and one that runs in seconds per rule.

## Choosing a rung

The decision frame I use is deliberately boring. Optimize in place when the workload is FAQ-shaped and the misses are occasional; it costs days and fixes only part of the problem. Add Azure AI Search and Azure OpenAI behind the agent when documents are long, numerous, or rule-checked; this is the recommended default for real document workloads, it costs a few weeks of engineering and a monthly Azure bill in the low hundreds of dollars, and it preserves everything that already works, including intake, Dataverse, and the Copilot Studio shell. Rebuild fully on Azure only when the organization has decided to leave Power Platform anyway. Replacing components that already work, for the sake of architectural purity, rarely pays off.

One adjacent lesson that saves a day of debugging on the Tier 3 path: Azure Functions cannot authenticate to Dataverse with function keys. The working pattern is an Entra ID app registration mapped to a Dataverse application user holding a least-privilege security role.

## Caveats

The documented numbers above were verified against Microsoft Learn on June 10, 2026, and these limits move; treat the quotas page as the source of truth and this post as the map of which limits matter. The quality thresholds are field observations, not specifications. And nothing here argues against Copilot Studio. It is the right front door for almost every conversational workload on the Microsoft stack, including the document ones, provided the reading happens somewhere built for reading.

I wrote earlier about [which Copilot Studio knowledge sources honor your permission model](https://jakubbjl.github.io/2026/06/10/security-trimming-copilot-studio-knowledge-sources.html); that post maps the security dimension of the same decision this one maps for capacity. Pick a knowledge source and you have picked both.

Copilot Studio did not fail to read your documents. It was never the reader. Put a retrieval pipeline behind it and let the front door be a front door.

## References

- [Copilot Studio quotas and limits](https://learn.microsoft.com/microsoft-copilot-studio/requirements-quotas), Microsoft Learn
- [Generative answers pointing to SharePoint sources don't return results: file size support](https://learn.microsoft.com/troubleshoot/power-platform/copilot-studio/generative-answers/sharepoint-no-response), Microsoft Learn
- [Unstructured data as a knowledge source](https://learn.microsoft.com/microsoft-copilot-studio/knowledge-unstructured-data), Microsoft Learn
- [Connect to custom knowledge sources](https://learn.microsoft.com/microsoft-copilot-studio/guidance/custom-knowledge-sources), Microsoft Learn
- [Add Azure AI Search as a knowledge source](https://learn.microsoft.com/microsoft-copilot-studio/knowledge-azure-ai-search), Microsoft Learn
