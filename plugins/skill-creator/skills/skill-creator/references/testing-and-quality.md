# Testing and Quality

## Table of Contents

1. [Three-Level Testing Methodology](#three-level-testing-methodology)
2. [Level 1: Triggering Tests](#level-1-triggering-tests)
3. [Level 2: Functional Tests](#level-2-functional-tests)
4. [Level 3: Performance Tests](#level-3-performance-tests)
5. [Success Criteria](#success-criteria)
6. [Pre-Release Checklist](#pre-release-checklist)

---

## Three-Level Testing Methodology

Test skills at three levels of increasing depth. Each level catches different categories of issues.

| Level | What It Tests | How to Test | Time |
|-------|--------------|-------------|------|
| 1. Triggering | Does the skill activate for the right prompts? | Send test prompts to Claude | 5 min |
| 2. Functional | Does the skill produce correct output? | Run through real use cases | 15-30 min |
| 3. Performance | Does the skill handle edge cases and scale? | Stress test with complex inputs | 30+ min |

## Level 1: Triggering Tests

Verify the skill activates when it should and stays silent when it shouldn't.

### Positive triggering (should activate)

Test 5-8 prompts that represent the skill's core use cases:

```
Prompt: "Create a new PDF from this data"
Expected: pdf-editor skill triggers
Result: ✅ / ❌

Prompt: "Help me merge these two documents"
Expected: pdf-editor skill triggers
Result: ✅ / ❌
```

### Negative triggering (should NOT activate)

Test 3-5 prompts that are adjacent but outside scope:

```
Prompt: "What's the weather today?"
Expected: skill does NOT trigger
Result: ✅ / ❌

Prompt: "Write me a Python script"
Expected: skill does NOT trigger (unless it's a coding skill)
Result: ✅ / ❌
```

### Ambiguous triggering

Test 2-3 edge-case prompts:

```
Prompt: "Convert this file"
Expected: May or may not trigger depending on context
Note: Acceptable either way, but document behavior
```

### Common triggering failures

- **Never triggers**: Description is too vague or uses jargon Claude doesn't associate with user language
- **Always triggers**: Description is too broad, overlapping with common tasks
- **Wrong skill triggers**: Two skills have overlapping descriptions

## Level 2: Functional Tests

Run the skill through its core workflows with real inputs.

### Test matrix

For each major use case, test with:

1. **Happy path**: Standard input, expected format, no complications
2. **Minimal input**: Bare minimum the user might provide
3. **Rich input**: Detailed request with many parameters or constraints
4. **Edge cases**: Empty files, very large inputs, unusual formats

### What to verify

- [ ] Output matches the expected format from SKILL.md templates
- [ ] All referenced scripts run without errors
- [ ] All referenced files (references/, assets/) are found and loaded correctly
- [ ] Error messages are clear and actionable when things fail
- [ ] The skill doesn't hallucinate tools or capabilities it doesn't have

### Script testing

For each included script:

```bash
# Test with expected input
python scripts/my_script.py test_input.txt
# Verify: exit code 0, output matches expected

# Test with missing input
python scripts/my_script.py nonexistent.txt
# Verify: clear error message, non-zero exit code

# Test with malformed input
python scripts/my_script.py malformed_input.txt
# Verify: graceful handling, helpful error message
```

## Level 3: Performance Tests

For skills that handle complex or large-scale tasks.

### Context window pressure

- Test with inputs that are 50%+ of typical context window
- Verify the skill degrades gracefully (summarizes, paginated, etc.)
- Check that progressive disclosure works (references loaded only when needed)

### Multi-step reliability

- Run the full workflow 3 times end-to-end
- Verify consistent results across runs
- Check that intermediate failures don't corrupt the final output

### Concurrent skill interaction

- Test with other skills active simultaneously
- Verify no namespace collisions or interference
- Check that the skill's description doesn't steal triggers from other skills

## Success Criteria

### Quantitative

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Trigger accuracy | 90%+ correct activations | Run 10+ test prompts |
| Happy path success | 100% for core use cases | Run each main workflow |
| Script reliability | 0 unhandled crashes | Run all scripts with varied input |
| Context efficiency | SKILL.md under 500 lines | Line count |

### Qualitative

- [ ] Output quality is visibly better than Claude without the skill
- [ ] The skill handles errors without confusing the user
- [ ] Instructions are unambiguous (no "it depends" without guidance)
- [ ] Reference files are loaded only when needed (progressive disclosure works)

## Pre-Release Checklist

### Before first release

- [ ] All Level 1 triggering tests pass
- [ ] All Level 2 functional tests pass for core use cases
- [ ] `quick_validate.py` passes with no errors
- [ ] SKILL.md is under 500 lines
- [ ] Description is under 1024 characters
- [ ] All file paths in SKILL.md are correct (relative paths, case-sensitive)
- [ ] No TODO or placeholder text remains in any file
- [ ] Scripts have been tested with at least happy-path inputs
- [ ] `package_skill.py` produces a valid .skill file

### During iteration

- [ ] Re-run triggering tests after description changes
- [ ] Re-run functional tests after workflow changes
- [ ] Verify reference file paths still valid after reorganization
- [ ] Check that new content doesn't push SKILL.md over 500 lines

### Before distribution

- [ ] Test in a clean environment (no leftover state from development)
- [ ] Verify the .skill file unpacks correctly
- [ ] Test the packaged skill from scratch (not from development environment)
- [ ] Document any environment requirements in `compatibility` field
