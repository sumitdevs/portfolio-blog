---
title: "I Added a Context-Aware AI Feature Without a Machine Learning Course"
description: "How I built a context-aware AI Dictionary feature using embeddings, vector search, Node.js, PostgreSQL, and a Chrome extension without a machine learning background."
date: "2026-03-10"
author: "Sumit"
tags: ["AI", "Vector Search", "Node.js", "PostgreSQL", "Chrome Extension"]
image: "https://ik.imagekit.io/hk2o7q2xu/posts/post1.jpg"
---

# I Added a Context-Aware AI Feature Without a Machine Learning Course

I want to start with the honest version: I had no idea what I was doing when I started building the AI Dictionary.

I knew I wanted the Chrome extension to return context-aware word meanings, not the same dictionary definition regardless of what you are reading. But I had no background in machine learning, had not touched Python, and had zero experience with embeddings or vector databases.

What I did have was a MERN stack background, a working understanding of REST APIs, and enough stubbornness to figure things out as I went.

Turns out that is enough.

## The Problem I Was Actually Trying to Solve

Standard dictionary APIs return the same result for "bank" every time. Whether you are reading about rivers or finance, you get a generic definition. That is fine for most use cases, but it is not how humans think about meaning.

When you highlight a word while reading something, you already have context. The paragraph around that word tells you almost everything you need to understand it. I wanted the extension to use that context and pick the meaning that actually fits what you are reading.

That meant semantic search.

Comparing meaning, not keywords.

## What I Thought Embeddings Were vs. What They Actually Are

I had heard the word "embeddings" thrown around in AI circles and imagined something complicated. The actual concept is simpler than it sounds.

An embedding is just a list of numbers that represents the meaning of a piece of text. Similar texts have similar numbers. That is it. You convert sentences into these number arrays and then find the ones that are mathematically closest to each other.

So if I embed the sentence "the bank of the river was muddy" and compare it against embedded definitions, the river-related meanings will be numerically closer than the finance-related ones.

That is the whole trick.

## The Tech Stack I Ended Up With

Here is what actually runs under the hood:

- **Xenova/Transformers**: a JavaScript port of Hugging Face Transformers that runs in the browser. I use this to generate embeddings client-side, which means no extra API call just for the embedding step.
- **PostgreSQL with pgvector**: a Postgres extension that lets you store and query vectors using similarity search. Fast, with no separate vector database service needed.
- **Node.js + Express**: a REST API that receives the word and surrounding context, queries pgvector, and returns the best-matching definitions.
- **Chrome Extension API**: the content script captures selected text and the surrounding paragraph, then calls the backend.

The flow looks like this:

1. User highlights a word.
2. The extension grabs the word and context paragraph.
3. The extension sends that data to my API.
4. The API generates an embedding for the context.
5. pgvector searches for the most similar pre-embedded definitions.
6. The API returns the best match.

## The Part Nobody Tells You About pgvector

Setting up the pgvector extension itself is straightforward. The tricky part is pre-embedding your definitions.

You need to run all your dictionary entries through the embedding model and store those vectors in the database before any queries can run.

That is a one-time process, but it takes longer than you might expect if you have a lot of entries. I ran it as a Node.js script and just let it run in the background. Not complicated, just something you need to plan for.

The query itself, once everything is embedded, is genuinely fast. Under a second for most requests. PostgreSQL's vector similarity operators are well-optimized.

```sql
SELECT word, definition
FROM definitions
ORDER BY embedding <=> $1
LIMIT 3;
```

That `<=>` operator is cosine distance. The lower the distance, the more similar the meanings. You just pass the query embedding as `$1`, and Postgres handles the rest.

## What Went Wrong First

My initial setup was using the embedding model server-side. Every time the extension made a request, the backend would load the model, generate the embedding, and then query the database.

This was slow. Around 3 to 4 seconds per request, which completely killed the user experience.

Moving the embedding generation to the client side using Xenova/Transformers cut that down dramatically. The model downloads once when the extension is first used, then runs locally in the browser for every subsequent request.

The backend only needs to handle the vector query, which is fast.

This also means less compute on the server, which matters for hosting costs.

## What Actually Surprised Me

How little ML knowledge you actually need to build something useful with this approach.

I did not train a model. I did not touch Python or PyTorch. I used a pre-trained transformer model through a JavaScript library and a database extension that does the math for me.

The value comes from knowing how to wire these pieces together into a working system, which is a software engineering skill, not an ML research skill.

That said, I do think understanding why embeddings work helps you make better decisions about model selection, similarity thresholds, and edge cases. There is a difference between knowing how to use a tool and understanding what it is doing.

I am firmly in the first camp for now, but I am working on the second.

## If You Want to Build Something Similar

Start small. Pick a single, well-defined use case.

My advice:

- Use `@xenova/transformers` in the browser for embedding generation. It removes a whole network call from your critical path.
- Use pgvector with a Postgres database you are already running. You do not need a separate vector database service at smaller scales.
- Pre-embed your reference data as a one-time setup script.
- Keep your context window small. One or two sentences around the selected word is plenty.

The first version does not have to be perfect. Mine worked in 200ms when I finally got the architecture right, and that felt like magic compared to the 4-second first attempt.

The actual ML is less mysterious than it looks from the outside. The engineering challenges, such as performance, data modeling, and API design, are where the real work is.

Which is good news, because that is the part I know how to do.

