# Ticket Agent System — Claude Code Instructions

## What we are building
An agentic service ticket resolver. Full architecture is in docs/.
Read ALL files in docs/ before writing any code.
Start with docs/00-overview.md, then read in order 01 through 13.

## Architecture principle
Agent = Model + Harness. 
The LLM only reasons and plans. 
The Harness controls everything else.
The Orchestrator is the only thing that transitions state.
The Policy Guard is the only thing that dispatches tool calls.
The LLM never calls anything directly.

## Tech stack (to be confirmed but start with this)
- Language: TypeScript (Node.js)
- Orchestrator: custom state machine (no LangChain, no LangGraph)
- Database: PostgreSQL + pgvector (episodic memory)
- Cache: Redis (working memory)
- Search: Elasticsearch or Azure AI Search (KB)
- Graph: Neo4j (knowledge graph)
- Queue: BullMQ (agent job queue)
- LLM: Anthropic Claude via SDK
- MCP: custom MCP gateway
- UI: Next.js + React

## What NOT to do
- Do not use LangChain or LangGraph
- Do not use AutoGen or CrewAI
- Do not let the LLM call tools directly
- Do not store credentials in code
- Do not write freeform LLM output parsing — always use structured output
- Do not skip the policy guard layer
- Do not build the UI before the orchestrator works

## Build order
See BUILDPLAN.md for the exact sequence.