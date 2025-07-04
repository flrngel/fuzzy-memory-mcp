# Knowledge Graph Memory Server (from [official site](https://github.com/modelcontextprotocol/servers/tree/main/src/memory) enhanced with Fuzzy Search)

A basic implementation of persistent memory using a local knowledge graph. This lets Claude remember information about the user across chats. This version has been enhanced with `fuse.js` to provide fuzzy, semantic searching capabilities.

## Instruction prompt. (Add this to your CLAUDE.md or any other related)
```
Here is an instruction guide for each tool, focusing on best practices for using the knowledge graph effectively.

### A Guide to Using Knowledge Graph Memory

This guide outlines best practices for interacting with your knowledge graph memory. Following these principles will help you build a clean, accurate, and useful memory over time. The core idea is to first **search** for what you know, then **act** to add, update, or remove information.

---

#### **`search_nodes`**

This is your primary tool for discovery. It performs a fuzzy search across all entity names, types, and observations to find relevant information.

*   **Best Practice:** Always search before you create. To avoid creating duplicate entities (e.g., "Jane_Doe" when "Jane_Doe_Dev" already exists), start with a broad search to see what the graph already knows.
*   **Invocation Tip:** Use conceptual queries. You don't need an exact name. A query like "project manager who likes dogs" will effectively search observations across all entities to find the best match. Review the returned `score` to understand the confidence of the match.

---

#### **`create_entities`**

Use this to establish a new person, place, organization, or concept as a node in your graph.

*   **Storage Tip:** Choose a consistent and unique `name` for each entity (e.g., `FirstName_LastName`, `Project_Name`). This name is the permanent identifier.
*   **Invocation Tip:** Create entities with a few core `observations` from the start. An entity is more useful when it's created with initial facts, such as "is a software engineer" or "founded in 2021".

---

#### **`add_observations`**

Use this to add new facts or attributes to an entity that already exists.

*   **Storage Tip:** Keep observations atomic. Each observation should represent a single, discrete fact (e.g., use "Loves hiking" and "Lives in Colorado" as two separate observations, not one). This makes information easier to manage and remove later.
*   **Invocation Tip:** This tool is for enriching existing entities. It will not add duplicate observations, so you can safely call it with a list of facts without worrying about creating redundant entries.

---

#### **`create_relations`**

This tool connects two existing entities with a directed, active-voice relationship (e.g., `Jane_Doe` -> `reports_to` -> `John_Smith`).

*   **Storage Tip:** Ensure both the `from` and `to` entities already exist before creating a relation between them. A relation is meaningless without its nodes.
*   **Invocation Tip:** Use a consistent vocabulary for `relationType` (e.g., always use `works_at`, not a mix of `works_at` and `employed_by`). This makes the graph structure predictable and easier to query.

---

#### **`open_nodes`**

Use this to retrieve one or more entities by their exact name, along with any relations that exist between them.

*   **Best Practice:** Use this when you know the exact name of an entity and want to see its details and local connections. It's more precise than `search_nodes` for targeted lookups.
*   **Invocation Tip:** Before updating or deleting, use `open_nodes` to inspect the entity and its relationships. This helps confirm you are targeting the correct information.

---

#### **`delete_observations`**

This tool removes specific facts from an entity.

*   **Best Practice:** This is the standard way to update an entity when a fact is no longer true (e.g., removing "is learning Spanish" after proficiency is achieved).
*   **Invocation Tip:** You must provide the *exact* text of the observation to be deleted. Use `open_nodes` first to retrieve the exact phrasing if you are unsure.

---

#### **`delete_relations`**

This tool removes a specific connection between two entities, leaving the entities themselves intact.

*   **Best Practice:** Use this to update the graph when a relationship changes. For example, if a person moves to a new team, you would delete their old `reports_to` relation.
*   **Invocation Tip:** To be successful, the call must exactly match the `from` entity, `to` entity, and `relationType` of the stored relation.

---

#### **`delete_entities`**

This is a destructive action that permanently removes an entity and all relations connected to it.

*   **Best Practice:** Be certain before using this tool. Deleting an entity causes a cascading delete of all its connections. If you only want to remove an incorrect fact, use `delete_observations` instead.
*   **Invocation Tip:** The tool will not fail if the entity doesn't exist, so you don't need to check for its existence before calling.

---

#### **`read_graph`**

This tool retrieves the entire knowledge graph—every entity and every relation.

*   **Best Practice:** Use this tool sparingly, as it can return a very large amount of data. It is best suited for offline analysis, debugging, or getting a complete overview of your memory.
*   **Invocation Tip:** For nearly all interactive tasks, prefer the more focused `search_nodes` or `open_nodes` tools for better performance and relevance.
```

## Core Concepts

### Entities
Entities are the primary nodes in the knowledge graph. Each entity has:
- A unique name (identifier)
- An entity type (e.g., "person", "organization", "event")
- A list of observations

Example:
```json
{
  "name": "John_Smith",
  "entityType": "person",
  "observations": ["Speaks fluent Spanish"]
}
```

### Relations
Relations define directed connections between entities. They are always stored in active voice and describe how entities interact or relate to each other.

Example:
```json
{
  "from": "John_Smith",
  "to": "Anthropic",
  "relationType": "works_at"
}
```
### Observations
Observations are discrete pieces of information about an entity. They are:

- Stored as strings
- Attached to specific entities
- Can be added or removed independently
- Should be atomic (one fact per observation)

Example:
```json
{
  "entityName": "John_Smith",
  "observations": [
    "Speaks fluent Spanish",
    "Graduated in 2019",
    "Prefers morning meetings"
  ]
}
```

## API

### Tools
- **create_entities**
  - Create multiple new entities in the knowledge graph
  - Input: `entities` (array of objects)
    - Each object contains:
      - `name` (string): Entity identifier
      - `entityType` (string): Type classification
      - `observations` (string[]): Associated observations
  - Ignores entities with existing names

- **create_relations**
  - Create multiple new relations between entities
  - Input: `relations` (array of objects)
    - Each object contains:
      - `from` (string): Source entity name
      - `to` (string): Target entity name
      - `relationType` (string): Relationship type in active voice
  - Skips duplicate relations

- **add_observations**
  - Add new observations to existing entities
  - Input: `observations` (array of objects)
    - Each object contains:
      - `entityName` (string): Target entity
      - `contents` (string[]): New observations to add
  - Returns added observations per entity
  - Fails if entity doesn't exist

- **delete_entities**
  - Remove entities and their relations
  - Input: `entityNames` (string[])
  - Cascading deletion of associated relations
  - Silent operation if entity doesn't exist

- **delete_observations**
  - Remove specific observations from entities
  - Input: `deletions` (array of objects)
    - Each object contains:
      - `entityName` (string): Target entity
      - `observations` (string[]): Observations to remove
  - Silent operation if observation doesn't exist

- **delete_relations**
  - Remove specific relations from the graph
  - Input: `relations` (array of objects)
    - Each object contains:
      - `from` (string): Source entity name
      - `to` (string): Target entity name
      - `relationType` (string): Relationship type
  - Silent operation if relation doesn't exist

- **read_graph**
  - Read the entire knowledge graph
  - No input required
  - Returns complete graph structure with all entities and relations

- **search_nodes**
  - Performs a fuzzy semantic search for nodes in the knowledge graph using Fuse.js
  - Input: `query` (string)
  - Searches across:
    - Entity names
    - Entity types
    - Observation content
  - Returns an array of search results, each containing:
    - `entity`: The matched entity object
    - `score`: Confidence score from 0.0 to 1.0 (higher is better)
  - Uses fuzzy matching with:
    - Threshold: 0.6 (0.0 = perfect match, 1.0 = matches anything)
    - Minimum match character length: 2
    - Location-independent matching

- **open_nodes**
  - Retrieve specific nodes by name
  - Input: `names` (string[])
  - Returns:
    - Requested entities
    - Relations between requested entities
  - Silently skips non-existent nodes

# Usage with Claude Desktop

### Setup

Add this to your claude_desktop_config.json:

#### NPX
```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": [
        "-y",
        "github:flrngel/fuzzy-memory-mcp#main"
      ]
    }
  }
}
```

#### NPX with custom setting

The server can be configured using the following environment variables:

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": [
        "-y",
        "github:flrngel/fuzzy-memory-mcp#main"
      ],
      "env": {
        "MEMORY_FILE_PATH": "/path/to/custom/memory.json"
      }
    }
  }
}
```

- `MEMORY_FILE_PATH`: Path to the memory storage JSON file (default: `memory.json` in the server directory)

# VS Code Installation Instructions

For quick installation, use one of the one-click installation buttons below:

[![Install with NPX in VS Code](https://img.shields.io/badge/VS_Code-NPM-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=memory&config=%7B%22command%22%3A%22npx%22%2C%22args%22%3A%5B%22-y%22%2C%22github%3Aflrngel%2Ffuzzy-memory-mcp%23main%22%5D%7D) [![Install with NPX in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-NPM-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=memory&config=%7B%22command%22%3A%22npx%22%2C%22args%22%3A%5B%22-y%22%2C%22github%3Aflrngel%2Ffuzzy-memory-mcp%23main%22%5D%7D&quality=insiders)

[![Install with Docker in VS Code](https://img.shields.io/badge/VS_Code-Docker-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=memory&config=%7B%22command%22%3A%22docker%22%2C%22args%22%3A%5B%22run%22%2C%22-i%22%2C%22-v%22%2C%22claude-memory%3A%2Fapp%2Fdist%22%2C%22--rm%22%2C%22mcp%2Fmemory%22%5D%7D) [![Install with Docker in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-Docker-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=memory&config=%7B%22command%22%3A%22docker%22%2C%22args%22%3A%5B%22run%22%2C%22-i%22%2C%22-v%22%2C%22claude-memory%3A%2Fapp%2Fdist%22%2C%22--rm%22%2C%22mcp%2Fmemory%22%5D%7D&quality=insiders)

For manual installation, add the following JSON block to your User Settings (JSON) file in VS Code. You can do this by pressing `Ctrl + Shift + P` and typing `Preferences: Open Settings (JSON)`.

Optionally, you can add it to a file called `.vscode/mcp.json` in your workspace. This will allow you to share the configuration with others. 

> Note that the `mcp` key is not needed in the `.vscode/mcp.json` file.

#### NPX

```json
{
  "mcp": {
    "servers": {
      "memory": {
        "command": "npx",
        "args": [
          "-y",
          "github:flrngel/fuzzy-memory-mcp#main"
        ]
      }
    }
  }
}
```

### System Prompt

The prompt for utilizing memory depends on the use case. Changing the prompt will help the model determine the frequency and types of memories created.

Here is an example prompt for chat personalization. You could use this prompt in the "Custom Instructions" field of a [Claude.ai Project](https://www.anthropic.com/news/projects). 

```
Follow these steps for each interaction:

1. User Identification:
   - You should assume that you are interacting with default_user
   - If you have not identified default_user, proactively try to do so.

2. Memory Retrieval:
   - Always begin your chat by saying only "Remembering..." and retrieve all relevant information from your knowledge graph
   - Always refer to your knowledge graph as your "memory"

3. Memory
   - While conversing with the user, be attentive to any new information that falls into these categories:
     a) Basic Identity (age, gender, location, job title, education level, etc.)
     b) Behaviors (interests, habits, etc.)
     c) Preferences (communication style, preferred language, etc.)
     d) Goals (goals, targets, aspirations, etc.)
     e) Relationships (personal and professional relationships up to 3 degrees of separation)

4. Memory Update:
   - If any new information was gathered during the interaction, update your memory as follows:
     a) Create entities for recurring organizations, people, and significant events
     b) Connect them to the current entities using relations
     b) Store facts about them as observations
```

## Building and Development

### Prerequisites
- Node.js and npm
- Docker (for building the Docker image)

### Local Development
If you've cloned this repository and want to run the server locally for development:

1.  Install dependencies (this will include `fuse.js` for fuzzy searching):
    ```sh
    npm install
    ```
2.  Compile and run the server:
    ```sh
    npm start
    ```

### Building the Docker Image

The `Dockerfile` handles installing all necessary dependencies.

```sh
docker build -t mcp/memory . 
```

## License

This MCP server is licensed under the MIT License. This means you are free to use, modify, and distribute the software, subject to the terms and conditions of the MIT License. For more details, please see the LICENSE file in the project repository.
