---
title: MCP tools
description: How Veilgate exposes HTTP APIs as MCP tools for LLM agents.
---

# MCP tools

Veilgate provides first‑class integration with the Model Context Protocol (MCP).

Key ideas:

- **MCP tools** wrap HTTP APIs behind Veilgate and expose them to LLM agents.
- Tools have descriptors (name, description, input/output schemas).
- A tool typically maps to one or more gateway routes by ID.

The Veilgate CLI includes scaffolding to generate an MCP server that:

- calls Veilgate via HTTP,
- uses the same auth, rate limit and observability stack as regular clients.

See the MCP how‑to guides and reference for details on configuring MCP tools.


