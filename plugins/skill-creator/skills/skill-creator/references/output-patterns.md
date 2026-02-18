# Output Patterns

Use these patterns when skills need to produce consistent, high-quality output.

## Template Pattern

Provide templates for output format. Match the level of strictness to your needs.

**For strict requirements (like API responses or data formats):**

```markdown
## Report structure

ALWAYS use this exact template structure:

# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data
- Finding 3 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```

**For flexible guidance (when adaptation is useful):**

```markdown
## Report structure

Here is a sensible default format, but use your best judgment:

# [Analysis Title]

## Executive summary
[Overview]

## Key findings
[Adapt sections based on what you discover]

## Recommendations
[Tailor to the specific context]

Adjust sections as needed for the specific analysis type.
```

## Examples Pattern

For skills where output quality depends on seeing examples, provide input/output pairs:

```markdown
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware

**Example 2:**
Input: Fixed bug where dates displayed incorrectly in reports
Output: fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation

Follow this style: type(scope): brief description, then detailed explanation.
```

Examples help Claude understand the desired style and level of detail more clearly than descriptions alone.

## Validation Pattern

When output must meet specific criteria, build validation into the workflow:

```markdown
## Output Validation

After generating the output, verify:

1. **Structural check**: All required sections are present
2. **Content check**: No placeholder text remains (search for TODO, TBD, FIXME)
3. **Format check**: Consistent heading levels, proper markdown syntax
4. **Length check**: Each section meets minimum/maximum word counts

If any check fails, fix the issue before presenting to user.
```

**When to use**: Skills producing structured documents, code, or data where correctness matters.

## Progressive Output Pattern

For long-running tasks, provide intermediate output to keep users informed:

```markdown
## Progress Reporting

For tasks with multiple steps, report progress:

1. After analysis: "Found X items to process. Starting with..."
2. At milestones: "Completed 5/20 files. Current results: ..."
3. On completion: Present full results with summary

Never go silent for more than one major step without updating the user.
```

**When to use**: Tasks involving many files, large datasets, or multi-step processing.

## Error Output Pattern

Define how errors should be communicated:

```markdown
## Error Reporting

When an operation fails:

1. **State what failed**: "Failed to parse config.yaml"
2. **State why**: "Line 42: Invalid YAML syntax - unexpected tab character"
3. **Suggest fix**: "Replace the tab with spaces on line 42"
4. **Offer alternatives**: "Or provide a corrected file and I'll retry"

Never report raw stack traces to users. Always translate to actionable guidance.
```

**When to use**: Any skill that interacts with external files, APIs, or processes that can fail.
