# Workflow Patterns

## Table of Contents

1. [Sequential Workflows](#sequential-workflows)
2. [Conditional Workflows](#conditional-workflows)
3. [Multi-MCP Coordination](#multi-mcp-coordination)
4. [Iterative Refinement](#iterative-refinement)
5. [Context-Aware Selection](#context-aware-selection)

---

## Sequential Workflows

For complex tasks, break operations into clear, sequential steps. Provide an overview of the process towards the beginning of SKILL.md:

```markdown
Filling a PDF form involves these steps:

1. Analyze the form (run analyze_form.py)
2. Create field mapping (edit fields.json)
3. Validate mapping (run validate_fields.py)
4. Fill the form (run fill_form.py)
5. Verify output (run verify_output.py)
```

**When to use**: Tasks with a fixed order where each step depends on the previous one.

**Best practice**: Number steps explicitly, name the scripts/actions, and describe expected output at each stage.

## Conditional Workflows

For tasks with branching logic, guide Claude through decision points:

```markdown
1. Determine the modification type:
   **Creating new content?** → Follow "Creation workflow" below
   **Editing existing content?** → Follow "Editing workflow" below

2. Creation workflow: [steps]
3. Editing workflow: [steps]
```

**When to use**: Tasks where the approach depends on user input, file type, or context.

**Best practice**: Place the decision tree early in the workflow so Claude can route correctly before reading variant-specific details.

## Multi-MCP Coordination

When a skill orchestrates multiple MCP tools together:

```markdown
## Data Pipeline Workflow

1. **Fetch data** using `mcp__datasource__query` tool
   - Build the query based on user request
   - Validate response contains expected fields

2. **Transform data** using local scripts
   - Run scripts/transform.py on the raw data
   - Handle missing or malformed fields gracefully

3. **Push results** using `mcp__destination__upload` tool
   - Format output to match destination schema
   - Confirm upload succeeded before reporting to user
```

**When to use**: Skills that bridge multiple MCP servers or combine MCP tools with local scripts.

**Best practice**: Clearly label which step uses which tool. Include validation between steps to catch failures early. Provide fallback instructions if an MCP tool is unavailable.

## Iterative Refinement

For tasks requiring multiple rounds of improvement:

```markdown
## Design Iteration Workflow

1. **Generate initial version**
   - Produce first draft based on user requirements
   - Present to user for review

2. **Collect feedback**
   - Ask specific questions about what to change
   - Categorize feedback: structural vs. cosmetic vs. content

3. **Apply refinements**
   - Address structural changes first
   - Then content changes
   - Then cosmetic polish

4. **Verify and repeat**
   - Present updated version
   - If user requests more changes, return to step 2
   - If approved, proceed to final output
```

**When to use**: Creative tasks (presentations, documents, designs), code generation, or any output that benefits from user feedback loops.

**Best practice**: Limit to 3-4 iteration rounds by default. Ask targeted questions rather than open-ended "what do you think?" to avoid vague feedback cycles.

## Context-Aware Selection

When a skill must choose between multiple approaches based on the environment:

```markdown
## Deployment Strategy Selection

Determine the target environment and select the appropriate workflow:

### Detection
- Check for `package.json` → Node.js project
- Check for `requirements.txt` or `pyproject.toml` → Python project
- Check for `Dockerfile` → Container-based deployment

### Node.js Projects
1. Run `npm ci` for clean install
2. Run `npm run build` for production build
3. Deploy using platform-specific method

### Python Projects
1. Create virtual environment
2. Install dependencies from lock file
3. Run migrations if applicable
4. Deploy using platform-specific method

### Container Projects
1. Build Docker image with appropriate tag
2. Push to container registry
3. Update deployment configuration
```

**When to use**: Skills that must adapt to different project types, frameworks, or environments.

**Best practice**: Put detection logic first, then branch to specific workflows. Keep each branch self-contained so Claude doesn't mix steps from different paths.
