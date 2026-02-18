# MCP Integration Patterns

Skills can leverage MCP (Model Context Protocol) servers to extend Claude's capabilities beyond local tools. This guide covers patterns for integrating MCP tools into skills effectively.

## Pattern 1: MCP Enhancement

The skill enhances an existing MCP tool with domain knowledge, workflows, or guardrails.

```markdown
# BigQuery Analyst

## Querying Data

Use the `mcp__bigquery__run-query` tool to execute SQL queries.

Before querying, always:
1. Read references/schema.md to understand table relationships
2. Check the user's question against known query patterns
3. Build the query using the schema documentation

After querying:
1. Format results as a readable table
2. Highlight key insights
3. Suggest follow-up analyses
```

**When to use**: When an MCP server provides raw capability (API access, database queries) but the skill adds context about *how* to use it well.

**Frontmatter**:
```yaml
allowed-tools:
  - mcp__bigquery__run-query
  - mcp__bigquery__list-tables
```

## Pattern 2: Multi-MCP Coordination

The skill orchestrates multiple MCP servers together.

```markdown
# Data Pipeline

## Workflow

1. **Extract**: Use `mcp__source-db__query` to pull raw data
2. **Transform**: Process data locally with scripts/transform.py
3. **Load**: Use `mcp__dest-api__upload` to push results
4. **Notify**: Use `mcp__slack__post-message` to alert the team

Each step must succeed before proceeding. On failure at any step,
report the error and suggest manual intervention.
```

**When to use**: When a workflow spans multiple external systems.

**Frontmatter**:
```yaml
allowed-tools:
  - mcp__source-db__query
  - mcp__dest-api__upload
  - mcp__slack__post-message
  - Bash
```

## Pattern 3: Fallback

The skill works with or without MCP tools, degrading gracefully.

```markdown
# Document Converter

## Converting Documents

### With MCP (preferred)
If `mcp__converter__convert` is available:
1. Upload source file using the MCP tool
2. Specify target format
3. Download converted result

### Without MCP (fallback)
If the MCP tool is unavailable:
1. Use local pandoc via Bash: `pandoc input.docx -o output.pdf`
2. If pandoc is not installed, ask the user to provide an alternative format
3. Report the limitation and suggest installing the MCP server
```

**When to use**: When the skill should work in environments where the MCP server may not be configured.

**Best practice**: Always check tool availability at the start of the workflow rather than failing mid-process.

## Pattern 4: Pre-flight Validation

The skill validates MCP tool availability and configuration before starting work.

```markdown
## Pre-flight Checks

Before starting the workflow, verify:

1. **Tool availability**: Check that `mcp__server__tool` is in the available tools list
2. **Authentication**: Run a lightweight test call (e.g., list tables, ping endpoint)
3. **Permissions**: Verify the tool has required access (read, write, admin)

If any check fails:
- Report which check failed and why
- Suggest configuration steps
- Offer alternative approaches if available

Do NOT proceed with the main workflow until all pre-flight checks pass.
```

**When to use**: For skills that depend heavily on MCP tools where failure mid-workflow would waste significant time or leave data in an inconsistent state.

## Declaring MCP Dependencies

### In frontmatter

```yaml
---
name: my-mcp-skill
description: Analyze data from the warehouse. Use when querying sales data, building reports, or exploring table schemas.
allowed-tools:
  - mcp__bigquery__run-query
  - mcp__bigquery__list-tables
  - mcp__bigquery__get-schema
---
```

### In SKILL.md body

Reference MCP tools by their full name: `mcp__server-name__tool-name`

Provide context for each tool:
```markdown
## Available Tools

- `mcp__bigquery__run-query`: Execute a SQL query and return results
- `mcp__bigquery__list-tables`: List all accessible tables in a dataset
- `mcp__bigquery__get-schema`: Get column definitions for a specific table
```

### Common mistakes

- **Guessing tool names**: Always verify exact tool names from the MCP server
- **Missing allowed-tools**: If the skill uses MCP tools, declare them in frontmatter
- **No fallback**: Skills that break completely without MCP are fragile
- **Assuming tool state**: Don't assume MCP tools maintain state between calls
