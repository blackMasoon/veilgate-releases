---
title: MCP configuration
description: Reference for configuring MCP tools in Veilgate.
---

# MCP configuration

MCP (Model Context Protocol) support in Veilgate is driven by configuration.

At a high level, MCP configuration describes:

- tools exposed to LLM agents,
- mappings from tools to gateway routes / HTTP endpoints,
- JSON schemas for tool inputs and outputs.

This document will be extended once the MCP configuration model is wired into
`GatewayConfig` and the admin API.


