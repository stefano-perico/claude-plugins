# Description Craft

The `description` field in YAML frontmatter is the most important piece of a skill. It determines when Claude activates the skill. A poorly written description means the skill never triggers or triggers for wrong tasks.

## Anatomy of a Good Description

A description should answer three questions:
1. **What does this skill do?** (capabilities)
2. **When should Claude use it?** (triggers)
3. **What specific inputs/outputs are involved?** (scope)

### Structure Pattern

```
[1-2 sentences: what the skill does]. Use when [specific trigger conditions].
[Optional: list specific scenarios with (1), (2), (3) numbering].
```

## Examples: Good vs. Bad

### Example 1: PDF Skill

**Bad:**
```yaml
description: Helps with PDF files
```
- Too vague. "Helps" is meaningless. Claude doesn't know when to trigger.

**Good:**
```yaml
description: Read, create, edit, merge, split, and annotate PDF files. Use this skill whenever the user wants to do anything with PDF files, including extracting text or tables, combining multiple PDFs, splitting pages apart, rotating pages, adding watermarks, creating new PDFs, or filling PDF forms.
```
- Lists specific capabilities. Enumerates trigger scenarios exhaustively.

### Example 2: Presentation Skill

**Bad:**
```yaml
description: Creates presentations
```
- Too short. Doesn't mention file format, editing, or reading.

**Good:**
```yaml
description: Use this skill any time a .pptx file is involved in any way - as input, output, or both. This includes creating slide decks, pitch decks, or presentations; reading, parsing, or extracting text from any .pptx file; editing existing presentations; and converting other content into slide format.
```
- Anchors to file type (.pptx). Covers input, output, and bidirectional use.

### Example 3: Internal Communications

**Bad:**
```yaml
description: Write internal documents for the company
```
- "Internal documents" is too broad. Could mean anything.

**Good:**
```yaml
description: Write internal communications using company formats. Use when asked to write status reports, leadership updates, 3P updates, company-wide announcements, team updates, quarterly reviews, or similar structured internal content.
```
- Lists specific document types. User-facing language.

## Three Categories of Use Cases

Structure triggers around these categories:

### 1. Document and Asset Skills
Trigger on file types or content formats:
```yaml
description: "...Use when .xlsx, .xlsm, .csv, or .tsv files are the primary input or output..."
```

### 2. Workflow Automation Skills
Trigger on task descriptions:
```yaml
description: "...Use when the user wants to deploy to AWS, set up CI/CD, or manage infrastructure..."
```

### 3. MCP Enhancement Skills
Trigger on tool-related context:
```yaml
description: "...Use when querying BigQuery tables, analyzing data schemas, or building SQL queries against the data warehouse..."
```

## The `allowed-tools` Field

The optional `allowed-tools` frontmatter field declares which tools the skill needs:

```yaml
---
name: my-data-skill
description: Query and analyze data from the company database...
allowed-tools:
  - mcp__bigquery__run-query
  - mcp__bigquery__list-tables
  - Bash
  - Read
---
```

### When to use `allowed-tools`

- **MCP tools**: Always declare MCP tools the skill depends on
- **Standard tools**: Declare if the skill has specific tool requirements (e.g., only `Read` + `Bash`, no file editing)
- **Restrictive skills**: Use to prevent the skill from using tools it shouldn't (e.g., a read-only analysis skill should not have `Write` or `Edit`)

### Best practices

- Only list tools the skill genuinely needs
- If a skill works with standard tools only (Read, Write, Edit, Bash, Glob, Grep), you can omit `allowed-tools`
- Always test that declared tools are actually available in the target environment

## Description Anti-Patterns

### Too much implementation detail
```yaml
# ❌ Don't explain HOW the skill works
description: Uses pdfplumber library to extract text, reportlab for creation, and PyPDF2 for merging PDF documents
# ✅ Explain WHAT and WHEN
description: Read, create, merge, and edit PDF files. Use when the user wants to work with PDF documents.
```

### Jargon the user won't use
```yaml
# ❌ Technical jargon
description: OOXML document manipulation with tracked revisions and content control elements
# ✅ User-facing language
description: Create and edit Word documents (.docx) with support for tracked changes, comments, and formatting
```

### Missing the "when to use" clause
```yaml
# ❌ No trigger guidance
description: A comprehensive toolkit for data visualization
# ✅ Clear trigger
description: Create data visualizations and charts. Use when the user asks for bar charts, line graphs, scatter plots, dashboards, or any visual representation of data.
```
