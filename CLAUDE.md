# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Model Context Protocol (MCP) server that provides persistent memory capabilities for Claude through a knowledge graph structure. The server manages entities, relations, and observations stored in JSON Lines format.

## Build & Development Commands

- **Install dependencies**: `npm install`
- **Build**: `npm run build` (compiles TypeScript to JavaScript)
- **Watch mode**: `npm run watch` (continuous compilation during development)
- **Run locally**: `npx . --help` or `node dist/index.js`

## Architecture

The codebase consists of a single TypeScript file (`index.ts`) that implements:

1. **KnowledgeGraphManager**: Core class handling all graph operations
   - Loads/saves graph data from/to JSONL file
   - Manages entities, relations, and observations
   - Implements search functionality

2. **MCP Server**: Exposes 9 tools for knowledge graph manipulation:
   - `create_entities`, `create_relations`, `add_observations`
   - `delete_entities`, `delete_relations`, `delete_observations`  
   - `read_graph`, `search_nodes`, `open_nodes`

3. **Data Storage**: Uses JSON Lines format (.jsonl) for efficient append operations
   - Default path: `memory.json` in the same directory as the script
   - Configurable via `MEMORY_FILE_PATH` environment variable

## Important Notes

- The package name was changed from `flrngel/fuzzy-memory-mcp` to `@flrngel/fuzzy-memory-mcp` to follow npm's scoped package naming convention
- The server implements fuzzy search using the fuse.js library with configurable threshold and matching parameters
- The server runs on stdio and is designed to be used with Claude Desktop or other MCP clients
- TypeScript configuration targets ES2022 with ESNext modules for modern JavaScript features