# ADR-0001: pgvector vs. Qdrant

- **Status:** accepted
- **Date:** 2026-11-20
- **Phase:** 0

## Context
The app will not only do "find similar text". Every search also has rules, and relationships:

- Only search collections the user is allowed to see.
- Only search the current version of each document.

So a real query reads: "Find the 5 nearest chunks, but only among chunks this user may see."

The rules (who is a member of which collection, which version is active) already live in Postgres. If vectors live in Qdrant, the two systems have to agree with each other. If vectors live in pgvector, one SQL query does both jobs.

## Options considered
1. **Option A: pgvector**: An extension in postgres database itself.
2. **Option B: Qdrant**: A dedicated server for storing vector data.

## Decision
Qdrant makes me choose between duplicate permission data and post-filtering. We picked pgvector because with it we can handle the atomicity between data changes easily because all the data lives in same table or DB. 

## Consequences
The cost is post-filtering, which we mitigate iterative scans. We'll know the limit is a problem if a query with a restrictive filter returns fewer than k results.

pgvector is weaker at huge scale (tens of millions of vectors) and has fewer vector-specific features. A personal knowledge base probably has tens of thousands of chunks, so we're far from that limit. That's a reason to pick pgvector now. It's also a trigger for when you'd revisit the decision.

## What I learned
- Vector DB and how it works
- pgvector vs. Qdrant
- post filtering and pre filtering
