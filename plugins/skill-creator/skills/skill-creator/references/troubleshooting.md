# Troubleshooting

## Table of Contents

1. [Skill Won't Upload or Install](#skill-wont-upload-or-install)
2. [Skill Doesn't Trigger](#skill-doesnt-trigger)
3. [Skill Triggers Too Often](#skill-triggers-too-often)
4. [Instructions Not Followed](#instructions-not-followed)
5. [MCP Tool Issues](#mcp-tool-issues)
6. [Large Context Problems](#large-context-problems)

---

## Skill Won't Upload or Install

### Symptoms
- Error during upload to Claude.ai
- Validation fails in Claude Code
- `.skill` file rejected

### Common Causes and Fixes

**Invalid frontmatter**
```
❌ Missing --- delimiters
❌ YAML syntax error (tabs instead of spaces)
❌ Missing required field (name or description)
```
Fix: Run `quick_validate.py` to identify the exact issue. Ensure frontmatter starts and ends with `---` on their own lines.

**Name validation failure**
```
❌ Name uses uppercase: "My-Skill"
❌ Name contains spaces: "my skill"
❌ Name starts/ends with hyphen: "-my-skill-"
❌ Name has consecutive hyphens: "my--skill"
❌ Name exceeds 64 characters
```
Fix: Use kebab-case only: `my-skill-name`, lowercase, letters/digits/hyphens.

**Description issues**
```
❌ Contains angle brackets: "Use for <type> files"
❌ Exceeds 1024 characters
❌ Contains TODO or placeholder text
```
Fix: Remove angle brackets, shorten description, replace placeholders with real content.

**Unknown frontmatter fields**
```
❌ version: 1.0  (not an allowed field)
❌ author: John   (not an allowed field)
```
Fix: Only these fields are allowed: `name`, `description`, `license`, `allowed-tools`, `metadata`, `compatibility`.

## Skill Doesn't Trigger

### Symptoms
- User asks a relevant question but the skill never activates
- Another skill activates instead

### Diagnosis Steps

1. **Check the description** - Is it specific enough? Claude uses the description to decide when to activate.
2. **Test with exact trigger phrases** - Use the exact wording from the description.
3. **Check for competing skills** - Another skill with overlapping description may be winning.

### Common Causes and Fixes

**Description too vague**
```
❌ "Helps with documents"
✅ "Create, edit, and analyze Word documents (.docx files). Use when the user wants to work with .docx files for creating documents, editing content, or extracting text."
```

**Description uses jargon users don't use**
```
❌ "OOXML document processing and manipulation"
✅ "Work with Word documents (.docx) - create, edit, read, and format"
```

**Missing trigger scenarios**
```
❌ "Create presentations"
✅ "Create presentations, slide decks, pitch decks. Use when .pptx files are involved as input or output, including creating, editing, or reading slides."
```

## Skill Triggers Too Often

### Symptoms
- Skill activates for unrelated requests
- Skill interferes with other skills or normal Claude behavior

### Common Causes and Fixes

**Description too broad**
```
❌ "Helps with any file processing task"
✅ "Process and manipulate PDF files specifically. Use for merging, splitting, rotating, or extracting text from PDF documents."
```

**Overlapping with common tasks**
```
❌ "Write and format text" (overlaps with normal Claude)
✅ "Write and format text within .docx Word documents using python-docx library"
```

**Fix**: Narrow the description to specific file types, tools, or domains. Add qualifying phrases like "Use when..." with concrete conditions.

## Instructions Not Followed

### Symptoms
- Claude loads the skill but doesn't follow its workflow
- Output doesn't match the templates or patterns in SKILL.md
- Steps are skipped or executed out of order

### Common Causes and Fixes

**Instructions are suggestions, not directives**
```
❌ "You might want to check the format first"
✅ "ALWAYS check the format first before processing"
```

**Too many instructions competing for attention**
```
❌ 800-line SKILL.md with dozens of detailed rules
✅ Under 500 lines, core workflow prominent, details in references/
```

**Ambiguous decision points**
```
❌ "Choose the best approach for the situation"
✅ "If the file is under 10 pages, use approach A. If over 10 pages, use approach B."
```

**Critical steps buried in text**
Use formatting to make key instructions stand out:
```markdown
**IMPORTANT**: Always run validate.py before generating output.

> ⚠️ Never skip the verification step - it catches 90% of formatting errors.
```

## MCP Tool Issues

### Symptoms
- Skill references MCP tools that aren't available
- MCP tool calls fail silently or with cryptic errors
- Skill works in development but fails in production

### Common Causes and Fixes

**MCP server not connected**
```
Symptom: "Tool mcp__server__tool not found"
Fix: Verify the MCP server is configured and running.
     Add fallback instructions in SKILL.md for when MCP is unavailable.
```

**Tool name mismatch**
```
❌ Skill references: mcp__myserver__get_data
   Actual tool name: mcp__my-server__getData
Fix: Check exact tool names from the MCP server documentation.
```

**Missing `allowed-tools` in frontmatter**
If the skill needs MCP tools, declare them:
```yaml
allowed-tools:
  - mcp__server-name__tool-name
  - mcp__server-name__other-tool
```

**Graceful degradation pattern**
```markdown
## Data Fetching

If `mcp__datasource__query` is available, use it to fetch data directly.
If the MCP tool is unavailable, ask the user to provide the data as a file
or paste it into the conversation.
```

## Large Context Problems

### Symptoms
- Skill becomes unreliable with long conversations
- Earlier instructions get "forgotten"
- Output quality degrades mid-task

### Common Causes and Fixes

**SKILL.md is too long**
```
Symptom: Instructions from early in SKILL.md are ignored
Fix: Keep SKILL.md under 500 lines. Move details to references/.
```

**Loading too many references at once**
```
Symptom: Claude reads 3+ reference files and loses track
Fix: Structure references so only 1-2 are needed per task.
     Add clear "when to read" guidance in SKILL.md.
```

**No progressive disclosure**
```
❌ Everything in SKILL.md, 1200 lines
✅ Core workflow in SKILL.md (300 lines) + 5 reference files loaded on demand
```

**Repetitive content across files**
```
Symptom: Same information in SKILL.md AND references/guide.md
Fix: Single source of truth. Keep summaries in SKILL.md, details in references/.
```
