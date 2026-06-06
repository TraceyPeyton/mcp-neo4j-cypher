# MCP Neo4j Cypher and Shared Memory Description

## MCP Neo4j Cypher

mcp-neo4j-cypher is a Model Context Protocol server that exposes Neo4j database access to MCP-compatible clients. It is implemented with FastMCP and the asynchronous Neo4j Python driver.

The server provides three main tools:

1. get_neo4j_schema: inspects the graph schema by calling APOC's apoc.meta.schema, then returns a cleaned JSON representation of labels, properties, indexes, counts, and relationships.
2. read_neo4j_cypher: executes read-only Cypher queries. Before running the query, it uses EXPLAIN to reject write queries. Results are sanitized to remove oversized list values and can be truncated to a configured token limit.
3. write_neo4j_cypher: executes Cypher write queries and returns Neo4j summary counters. This tool can be disabled with read-only mode.

The server supports stdio, http, and sse transports. For HTTP-style transports it adds CORS middleware and trusted-host protection. Configuration is read from command-line arguments first and environment variables second, with defaults for local Neo4j development. Important settings include Neo4j URI, username, password, database, namespace, transport, host, port, read timeout, read-only mode, response token limit, and schema sample size.

Tool names can be prefixed with a namespace, which allows multiple Neo4j MCP servers to be connected in the same client session without name collisions.

## Shared Memory Folder

The Shared Memory folder contains a separate Neo4j-backed memory, caching, and execution logging system. Its purpose is to preserve useful prompt outputs, reusable Cypher queries, persistent knowledge, web content, and execution metadata so repeated work can be answered from memory before recomputing or searching externally.

### Core prompt cache

shared_memory.py is the simple public interface. It exposes helper functions such as:

1. init_shared_memory
2. cache_prompt_output
3. get_cached_output
4. get_cache_stats
5. list_cached_prompts
6. clear_old_cache
7. delete_cached_prompt
8. decorator_with_cache

Internally it uses Neo4jPromptCache, Neo4jQueryManager, and ExecutionLogger. Calls initialize lazily on first use. When a prompt output is cached or retrieved, the system also records execution metadata.

neo4j_prompt_cache.py stores prompt-answer pairs as PromptCache nodes. Each prompt is keyed by a SHA-256 hash for exact lookup, and fuzzy matching uses Python's difflib.SequenceMatcher against recent high-hit prompts. Cached answers are stored as strings, with JSON serialization used when the answer is structured data.

The prompt cache creates these Neo4j structures:

1. Unique constraint on PromptCache.hash
2. Indexes on prompt text, category, creation time, and hit count
3. PromptCache nodes containing prompt text, answer, category, metadata, timestamps, and hit count

## Stored query manager

neo4j_query_manager.py stores reusable Cypher queries as Query nodes. Each saved query has a unique name, Cypher code, description, query type, creation time, and modification time.

The query manager can:

1. Save a named query
2. List stored queries
3. Retrieve a query by name
4. Execute a stored query with optional parameters
5. Delete a stored query

This is useful for keeping common graph operations in the database itself instead of rewriting them each time.

## Persistent shared knowledge

shared_memory_system.py defines Shared Memory System, a higher-level knowledge store built around Shared Memory nodes. These nodes store topic-oriented knowledge with content, category, source, confidence, metadata, access count, and relevance score.

The shared memory system can:

1. Store knowledge by topic and category
2. Recall the best matching memory before doing external work
3. Search memories by keyword
4. Link cached prompts to memory entries with REFERENCES relationships
5. Update confidence scores
6. Delete memory entries
7. Report aggregate memory insights

The same file also defines Intelligent Memory Lookup, which sketches a lookup hierarchy:

1. Check Shared Memory persistent knowledge
2. Check Prompt Cache cached answers
3. Search or execute stored Query entries
4. Optionally fall through to an external source

The external-search step is a placeholder rather than a complete implementation.

## Execution logging

execution_logger.py records prompt and web execution metadata to both Neo4j and a CSV file at Shared Memory/execution_logs.csv.

Execution records include:

1. Unique execution ID
2. Timestamp
3. Prompt and optional URL
4. Source, such as fresh or cache
5. Compute time in milliseconds
6. Token usage
7. Input token count
8. Model name
9. Estimated cost
10. Category

Neo4j logs are stored as Execution Log nodes. The module also provides retrieval and summary methods for execution analytics.

### Web and prompt cache extensions

Additional modules extend the same memory idea:

1. web_content_cache.py caches fetched or processed web content.
2. web_cache_integration.py integrates web caching with execution logging.
3. neo4j_prompt_cache.py and prompt_cache_manager.py provide prompt-cache management flows.
4. shared_memory_init.py, shared_memory_examples.py, execution_logger_examples.py, and integrated_cache_demo.py demonstrate setup and usage.

## Relationship between the two systems

mcp-neo4j-cypher is the MCP server that lets an AI client inspect and query Neo4j. The Shared Memory folder is an application-level memory layer that also uses Neo4j as its backing store. They are related because both operate on Neo4j, and the shared-memory documentation describes using cached results before running new MCP/Cypher work.

## Appendix: Project file links

### MCP Neo4j Cypher implementation

1. [src/mcp_neo4j_cypher/server.py](src/mcp_neo4j_cypher/server.py): FastMCP server setup, Neo4j tools, transports, CORS, trusted-host handling, and query execution.
2. [src/mcp_neo4j_cypher/utils.py](src/mcp_neo4j_cypher/utils.py): configuration parsing, environment-variable handling, value sanitization, token truncation, and helper utilities.
3. [src/mcp_neo4j_cypher/__init__.py](src/mcp_neo4j_cypher/__init__.py): command-line entry point and argument parser.

### Packaging and runtime configuration

1. [pyproject.toml](pyproject.toml): Python package metadata, dependencies, build backend, and console script definition.
2. [uv.lock](uv.lock): locked dependency versions for reproducible installs.
3. [Dockerfile](Dockerfile): container image definition for running the MCP server.
4. [docker-compose.yml](docker-compose.yml): local Neo4j and MCP server composition for development.
5. [server.json](server.json): server configuration metadata.
6. [manifest.json](manifest.json): packaged extension manifest and user configuration schema.
7. [Makefile](Makefile): project automation commands.
8. [inspector.sh](inspector.sh): local MCP inspector launch helper.
9. [test.sh](test.sh): test runner helper.

### Documentation and assets

1. [README.md](README.md): main usage, configuration, transport, security, and deployment documentation.
2. [CHANGELOG.md](CHANGELOG.md): release history.
3. [assets/images/text2cypher-process.png](assets/images/text2cypher-process.png): Text2Cypher workflow diagram used by the README.

### Tests

1. [tests/unit/test_utils.py](tests/unit/test_utils.py): unit tests for configuration parsing, token truncation, and utility behavior.
2. [tests/integration/conftest.py](tests/integration/conftest.py): integration-test fixtures and Neo4j test setup.
3. [tests/integration/test_server_tools_IT.py](tests/integration/test_server_tools_IT.py): integration tests for MCP server tools.
4. [tests/integration/test_http_transport_IT.py](tests/integration/test_http_transport_IT.py): integration tests for HTTP transport.
5. [tests/integration/test_sse_transport_IT.py](tests/integration/test_sse_transport_IT.py): integration tests for SSE transport.
6. [tests/integration/test_stdio_transport_IT.py](tests/integration/test_stdio_transport_IT.py): integration tests for STDIO transport.

### Shared Memory files

The Shared Memory modules described above, such as `shared_memory.py`, `neo4j_prompt_cache.py`, `neo4j_query_manager.py`, `execution_logger.py`, `shared_memory_system.py`, `web_content_cache.py`, and `web_cache_integration.py`, are not present in this `mcp-neo4j-cypher` project directory.
