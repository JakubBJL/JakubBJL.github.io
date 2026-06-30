---
layout: post
title: "Two connections, one decision: grounding a Copilot Studio agent on Azure AI Search"
date: 2026-06-24T00:00:00.000Z
tags:
  - copilot-studio
  - agents
  - architecture
  - azure
human-approved: true
---

When someone says "ground the agent on our documents," the fast reflex is Add knowledge, pick Azure AI Search, paste a key and an index name, and watch it answer in a few minutes. That path is real. It works. It is also the one that produces the vague, confidently wrong answers people then blame on the model.

The built-in connection and the custom one are not the easy version and the hard version of the same thing. They are two different decisions. The gap between them is where retrieval quality lives, and on some agents retrieval quality is not polish. It is correctness.

This is the build side of an argument I have made before, that a serious document agent has to move retrieval into Azure rather than attaching files to the agent and hoping. That earlier piece was the why. This is the how, and the one decision that the how turns on.

## The mental model, in four parts

Four resources do the work, and it helps to hold each one's job separately before wiring them together. Azure Storage is where the document files live. Azure OpenAI embeddings turn text into meaning vectors. Azure AI Search chunks the content, indexes it, and serves hybrid retrieval. Copilot Studio is the conversational front end that asks the questions and presents the answers.

Put every Azure resource in the same region. It cuts latency and avoids cross-region transfer cost, and the embedding model has to be available in that region anyway, which is the constraint that usually decides the region for you.

## Standing up the index

Three resources, in order.

Storage first. A standard general-purpose v2 account with a private blob container holds the source files. A small habit pays off later: you can use the forward slash inside blob names as virtual folders, which gives you something to filter on once the index is built.

Embeddings second. Deploy an embedding model in Azure OpenAI and write down three things the search service will ask for, the endpoint, a key, and the exact deployment name. The deployment name trips people up because it is the name you chose, not the model name. text-embedding-ada-002 is the most broadly available and the one Microsoft's own samples target. The newer text-embedding-3 models are the cost-efficient choice where the region supports them.

Search third, and here the tier is a real decision rather than a slider. The pricing tier decides whether you get the semantic ranker, and the semantic ranker is the feature the quality path depends on. The honest version of the tier story is more precise than "Free has no ranker," because Microsoft's own documentation says three slightly different things. The try-for-free guidance states the Free tier does not support semantic ranking. The tier-feature matrix says the ranker runs on Free but is not recommended for real workloads. The billing documentation says the free request allowance exists on all tiers, while the standard pay-as-you-go plan, the one you reach the moment you exceed the small free allowance, requires Basic or higher. Net for a build: treat Basic as the floor. Basic also unlocks managed identity, which the import wizard wants. Basic lists around 75 USD a month in many regions, and Microsoft's free-account guidance describes it as roughly a third of the 200 USD monthly trial credit, so call it on the order of 70 to 75 USD. Standard S1, several times that, is the production tier. Confirm the current rate for your region before you quote a number to anyone, because list prices move.

Then let the wizard do the assembly. The Import and vectorize data wizard runs the whole pipeline in one pass. Point it at the blob container and the embedding deployment, choose the RAG path, and it creates the data source, an index with both text and vector fields, a skillset that cracks the documents and vectorizes the chunks, and an indexer that runs it. Enable the semantic ranker in the wizard and pick a schedule, once for testing, hourly or daily for production so new blobs index on their own.

Verify by chunk count, not file count. A handful of documents should produce tens of index records, because each document is split into chunks. An index document count several times your file count is the signal the skillset chunked correctly. A count equal to the file count means it did not, and everything downstream will quietly underperform.

## The seam where a hand-built index fails

The wizard hides a requirement that you meet the moment you outgrow it. Chunk-level indexing, one document projected to many search records, only works when the skillset declares an index projection. The wizard writes that projection for you. The day you hand-build the skillset, which you will do for any non-trivial index that needs custom fields or metadata, chunk-level retrieval stops working without it. The chunks, their vectors, and their parent metadata do not land on the right records, and you get an index that looks populated and retrieves badly.

This is the first place a from-scratch build breaks silently. It is worth knowing before you leave the wizard behind, because nothing errors. The pipeline runs green and the answers are simply worse than they should be.

## Two doors, and the decision underneath them

With an index that retrieves well, there are two ways to connect it to the agent.

The built-in connection is the fast path. In Copilot Studio, Add knowledge, Azure AI Search, then key plus endpoint plus index name. It works in minutes. It also gives you no control over the query type, no filtering, and no reranking. Vague or off-topic answers here are the expected ceiling of the path, not a fault you can fix from inside it.

For casual question answering over a forgiving corpus, that ceiling is fine. The decision is whether your agent lives below it. The test is not how the demo feels. It is what a wrong answer costs. On an agent where completeness matters, a reviewer checking a document against requirements, a compliance lookup, anything where "not found" gets read as "not there," a missed passage is a silent false negative. That is the most damaging error this class of system makes, because it is invisible. The answer looks complete. The built-in path gives you no lever to reduce that error, which means the choice between the two doors is not a convenience tradeoff you revisit later. It is a correctness decision you make up front.

The custom connection is the control path, and it is where the one stumbling block lives. You build a topic with the OnKnowledgeRequested trigger, and you can only do it in code view. The visual designer does not expose the trigger at all, which is the single most common first-time dead end. The skeleton is small:

```yaml
kind: AdaptiveDialog
beginDialog:
  kind: OnKnowledgeRequested
  id: main
  intent: {}
  actions:
    # Call your search endpoint, transform the results,
    # and assign the result to System.SearchResults
inputType: {}
outputType: {}
```

A topic that uses this trigger acts as a custom knowledge source and feeds the agent's generative answers. It also gets system variables that ordinary topics do not. System.SearchQuery is the context-aware, rewritten query Copilot Studio builds from the conversation so far, and it is what you send to search rather than the raw user turn, so multi-turn context survives. Call Azure AI Search through its Power Platform connector with a semantic hybrid query, which combines keyword, vector, and the semantic reranker in one call. Set the search text to System.SearchQuery, the result count to ten or fifteen, and the index vector field and semantic configuration name. Going above fifteen is wasted, because Copilot Studio uses at most fifteen snippets to compose an answer, and that limit is shared across every knowledge topic the agent has.

Then the part that makes the control path worth the trouble, the quality gate. The semantic ranker returns a reranker score on every result. Filter on it before you hand the results back:

```
Set(
  System.SearchResults,
  ForAll(
    Filter(SearchOutput.results, rerankerScore > 2.5),
    { Content: snippet, ContentLocation: url, Title: title }
  )
)
```

The reranker score runs from 0 to 4. Microsoft's own rubric reads 2 as somewhat relevant, answering the question only partially, and 3 as relevant but missing the detail to be complete, so a 2.5 cutoff admits only passages the model rates above merely partial. Raise it for stricter answers, lower it to cast a wider net. Two honest caveats keep the gate from misleading you. Microsoft warns against making the threshold too granular, because the score distribution shifts slightly with infrastructure and with ranking-model updates, so 2.5 is a starting line and not a constant to defend to two decimals. And if you apply a scoring profile to the index, the score you get back is a boosted score that can exceed 4, so a plain greater-than-2.5 filter assumes you have not turned scoring-profile boosting on. Field names in the expression above depend on how your connector output is shaped, so treat it as the pattern, not a paste-in.

Worth a note for anyone building today: Copilot Studio also added a custom search action node, generally available since mid-2025, that runs a search against a knowledge source and saves the raw results to a variable without summarizing them. It is a lighter way to reach the same raw-data control for some flows. The OnKnowledgeRequested path remains the one to know, because it is what plugs custom retrieval directly into generative answers.

## The failures you will actually hit

A short catalogue, because each of these costs an afternoon the first time and a minute every time after.

Built-in and custom must not both point at the same index on one agent. The duplication makes the custom topic fail to fire, and you debug the code when the cause is the leftover built-in connection. Remove one.

A missing Import and vectorize button on the search service means the tier lacks integrated vectorization. Upgrade to Basic. An AsyncResponsePayloadTooLarge error means you are handing back too much, so lower the result count or raise the score threshold. A missing semantic ranker option means the service is on Free. None of these announce their real cause, which is why they belong on a list.

## Cost, briefly

A development footprint is cheap. Storage is under a dollar a month, embeddings run on the order of cents per million tokens and are mostly a one-time indexing charge, and AI Search at Basic is the floor at roughly 75 to 80 USD a month all in. Production is a different conversation, because the search tier, the model meter, and the licensing around the agent scale on different curves and bill on different logic. I have treated that running-cost picture on its own, since the worst way to be surprised by an agent's bill is to add the wrong two numbers together.

## Closing

The quick connection is not a smaller version of the real one. It proves the pipeline in an afternoon, and for plenty of agents that is the whole job. The custom path is what you reach for the moment a missed passage would become a wrong answer someone acts on.

So the question to settle first is not how to connect Azure AI Search. The wizard answers that. The question is whether retrieval quality is correctness for this agent. Answer it, and the door you need is obvious. Skip it, and you will ship the fast path, demo it well, and find out which kind of agent you built only when someone trusts an answer that was never there.