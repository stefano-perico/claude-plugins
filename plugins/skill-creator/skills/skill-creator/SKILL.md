---
name: skill-creator
description: Guide for creating, testing, troubleshooting, and distributing effective skills. Use when users want to create a new skill, update an existing skill, debug a skill that isn't working (won't trigger, triggers too often, instructions ignored), test a skill's quality, integrate MCP tools into a skill, or package and distribute a skill. Covers the full skill lifecycle from design through deployment.
license: Complete terms in LICENSE.txt
---

# Skill Creator

This skill provides comprehensive guidance for creating effective skills that extend Claude's capabilities.

## About Skills

Skills are modular, self-contained packages that extend Claude's capabilities by providing specialized knowledge, workflows, and tools. They transform Claude from a general-purpose agent into a specialized agent equipped with procedural knowledge that no model can fully possess.

### What Skills Provide

1. Specialized workflows - Multi-step procedures for specific domains
2. Tool integrations - Instructions for working with specific file formats or APIs
3. Domain expertise - Company-specific knowledge, schemas, business logic
4. Bundled resources - Scripts, references, and assets for complex and repetitive tasks

### Three Categories of Use Cases

**Document & Asset Skills**: Extend Claude's ability to work with specific file formats (PDF, DOCX, PPTX, XLSX). Triggered by file types or content formats.

**Workflow Automation Skills**: Codify multi-step processes (deployment, code review, data pipelines). Triggered by task descriptions.

**MCP Enhancement Skills**: Add domain knowledge on top of MCP tools (BigQuery schemas, API context, database relationships). Triggered by tool-related context. See [references/mcp-integration.md](references/mcp-integration.md) for patterns.

## Core Principles

### Concise is Key

The context window is a public good. Skills share it with system prompt, conversation history, other skills' metadata, and the actual user request.

**Default assumption: Claude is already very smart.** Only add context Claude doesn't already have. Challenge each piece: "Does Claude really need this?" and "Does this justify its token cost?"

Prefer concise examples over verbose explanations.

### Set Appropriate Degrees of Freedom

Match specificity to the task's fragility and variability:

- **High freedom (text-based instructions)**: Multiple approaches valid, decisions depend on context
- **Medium freedom (pseudocode/scripts with parameters)**: Preferred pattern exists, some variation acceptable
- **Low freedom (specific scripts, few parameters)**: Operations fragile, consistency critical, specific sequence required

Think of Claude exploring a path: a narrow bridge needs guardrails (low freedom), an open field allows many routes (high freedom).

### Anatomy of a Skill

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name + description required)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/          - Executable code (Python/Bash/etc.)
    ├── references/       - Documentation loaded into context as needed
    └── assets/           - Files used in output (templates, icons, fonts)
```

#### SKILL.md (required)

- **Frontmatter** (YAML): `name` and `description` fields (required), plus optional `license`, `metadata`, `allowed-tools`, `compatibility`. Only `name` and `description` are read by Claude to determine triggering.
- **Body** (Markdown): Instructions loaded AFTER the skill triggers.

For detailed guidance on writing effective descriptions, see [references/description-craft.md](references/description-craft.md).

#### Scripts (`scripts/`)

Executable code for tasks requiring deterministic reliability or frequently rewritten.

- **When to include**: Same code rewritten repeatedly or deterministic reliability needed
- **Benefits**: Token efficient, deterministic, may execute without loading into context

#### References (`references/`)

Documentation loaded as needed into context.

- **When to include**: Detailed information Claude should reference while working
- **Best practice**: If files are large (>10k words), include grep search patterns in SKILL.md
- **Avoid duplication**: Information should live in SKILL.md OR references, not both

#### Assets (`assets/`)

Files used in output, not loaded into context (templates, images, fonts, boilerplate).

#### What NOT to Include

Do not create: README.md, INSTALLATION_GUIDE.md, QUICK_REFERENCE.md, CHANGELOG.md, or any auxiliary documentation. The skill should contain only what an AI agent needs to do the job.

### Progressive Disclosure

Skills use a three-level loading system:

1. **Metadata** (name + description) - Always in context (~100 words)
2. **SKILL.md body** - When skill triggers (<5k words)
3. **Bundled resources** - As needed (unlimited)

Keep SKILL.md under 500 lines. Split content into separate files when approaching this limit. Reference split files clearly from SKILL.md with "when to read" guidance.

**Key principle:** Keep only core workflow and selection guidance in SKILL.md. Move variant-specific details to reference files.

**Avoid deeply nested references** - Keep references one level deep from SKILL.md. Structure files >100 lines with a table of contents.

## Skill Creation Process

1. Understand the skill with concrete examples
2. Plan reusable skill contents (scripts, references, assets)
3. Initialize the skill (run init_skill.py)
4. Edit the skill (implement resources and write SKILL.md)
5. Package the skill (run package_skill.py)
6. Test and iterate based on real usage

### Step 1: Understanding the Skill with Concrete Examples

Skip only when usage patterns are already clearly understood.

Clearly understand concrete examples of how the skill will be used. Ask targeted questions:

- "What functionality should the skill support?"
- "Can you give examples of how this skill would be used?"
- "What would a user say that should trigger this skill?"

Avoid overwhelming users with too many questions at once. Conclude when there is a clear sense of the functionality the skill should support.

### Step 2: Planning the Reusable Skill Contents

Analyze each concrete example by:

1. Considering how to execute on the example from scratch
2. Identifying what scripts, references, and assets would help when executing these workflows repeatedly

Example analyses:

- `pdf-editor` for "Help me rotate this PDF" → `scripts/rotate_pdf.py` (same code each time)
- `frontend-webapp-builder` for "Build me a todo app" → `assets/hello-world/` template
- `big-query` for "How many users logged in today?" → `references/schema.md` documentation

### Step 3: Initializing the Skill

Skip if the skill already exists and only needs iteration.

Always run `init_skill.py` for new skills:

```bash
scripts/init_skill.py <skill-name> --path <output-directory>
```

The script creates the skill directory with SKILL.md template, example resource directories, and TODO placeholders. Customize or remove generated files as needed.

### Step 4: Edit the Skill

The skill is created for another instance of Claude to use. Include information that would be beneficial and non-obvious.

#### Learn Proven Design Patterns

Consult these references based on your skill's needs:

- **Multi-step processes**: See [references/workflow-patterns.md](references/workflow-patterns.md) for sequential, conditional, multi-MCP, iterative, and context-aware workflows
- **Output formats or quality standards**: See [references/output-patterns.md](references/output-patterns.md) for template, example, validation, progressive, and error patterns
- **Description writing**: See [references/description-craft.md](references/description-craft.md) for trigger optimization and good/bad examples
- **MCP tool integration**: See [references/mcp-integration.md](references/mcp-integration.md) for enhancement, coordination, fallback, and pre-flight patterns
- **Testing methodology**: See [references/testing-and-quality.md](references/testing-and-quality.md) for three-level testing and checklists
- **Debugging issues**: See [references/troubleshooting.md](references/troubleshooting.md) for common problems and fixes
- **Packaging and sharing**: See [references/distribution.md](references/distribution.md) for distribution channels and versioning

#### Writing Effective Skills - Quick Reference

**Progressive disclosure**: Core workflow in SKILL.md, details in references/. Only load what's needed.

**Error handling**: Define how errors should be communicated. Use the Error Output pattern from output-patterns.md.

**Description craft**: The description is the #1 factor in whether a skill works. Include WHAT it does + WHEN to use it + specific triggers. Never put "when to use" info in the body - it's loaded too late.

**Imperative form**: Always write instructions as directives ("Run the script", "Check the output") not suggestions ("You might want to run...").

#### Start with Reusable Skill Contents

Begin implementation with the reusable resources: `scripts/`, `references/`, `assets/`. This may require user input (e.g., brand assets, templates, documentation).

Test added scripts by actually running them. If many similar scripts exist, test a representative sample.

Delete any unneeded example files from initialization.

#### Update SKILL.md

##### Frontmatter

Write YAML frontmatter with `name` and `description`:

- `name`: Kebab-case skill name
- `description`: Primary triggering mechanism. Include both what the skill does and specific triggers/contexts. Include ALL "when to use" information here, not in the body.

Example:
```yaml
description: Comprehensive document creation, editing, and analysis with support
  for tracked changes, comments, formatting preservation, and text extraction. Use
  when Claude needs to work with professional documents (.docx files) for creating
  new documents, modifying content, working with tracked changes, or any document tasks.
```

Do not include fields beyond `name`, `description`, `license`, `allowed-tools`, `metadata`, `compatibility`.

##### Body

Write instructions using imperative/infinitive form for using the skill and its bundled resources.

### Step 5: Packaging a Skill

Package into a distributable .skill file:

```bash
scripts/package_skill.py <path/to/skill-folder>
scripts/package_skill.py <path/to/skill-folder> ./dist  # custom output
```

The script validates first (frontmatter, naming, structure, description), then packages if validation passes.

For distribution options beyond .skill files, see [references/distribution.md](references/distribution.md).

### Step 6: Test and Iterate

After the initial build, test the skill systematically and iterate.

#### Quick Validation

Run structural validation:
```bash
python scripts/quick_validate.py <path/to/skill-folder>
```

#### Comprehensive Testing

For thorough testing, follow the three-level methodology in [references/testing-and-quality.md](references/testing-and-quality.md):

1. **Level 1 - Triggering**: Does the skill activate for the right prompts? (5 min)
2. **Level 2 - Functional**: Does the skill produce correct output? (15-30 min)
3. **Level 3 - Performance**: Does it handle edge cases and scale? (30+ min)

#### Success Indicators

- Trigger accuracy: 90%+ correct activations across 10+ test prompts
- Happy path: 100% success for core use cases
- Script reliability: 0 unhandled crashes
- Context efficiency: SKILL.md under 500 lines
- Output quality visibly better than Claude without the skill

#### Iteration Workflow

1. Use the skill on real tasks
2. Notice struggles or inefficiencies
3. Identify how SKILL.md or resources should be updated
4. Implement changes and test again

For troubleshooting specific issues, see [references/troubleshooting.md](references/troubleshooting.md).
