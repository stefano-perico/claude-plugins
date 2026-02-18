# Distribution

How to package, share, and deploy skills across different platforms.

## Packaging

### Creating a .skill file

Use `package_skill.py` to create a distributable `.skill` file:

```bash
python scripts/package_skill.py path/to/skill-folder
python scripts/package_skill.py path/to/skill-folder ./dist  # custom output dir
```

The script validates the skill first, then creates a zip archive with `.skill` extension.

### What gets packaged

Everything in the skill directory:
- `SKILL.md` (required)
- `scripts/` - all executable scripts
- `references/` - all reference documentation
- `assets/` - all asset files
- `LICENSE.txt` - license file (if present)

### Pre-packaging checklist

- [ ] Run `quick_validate.py` - no errors
- [ ] No TODO or placeholder text in any file
- [ ] All file references in SKILL.md are correct
- [ ] Scripts tested with representative inputs
- [ ] SKILL.md under 500 lines

## Distribution Channels

### Claude.ai (Web)

Upload the `.skill` file directly through the Claude.ai interface:
1. Generate the `.skill` file with `package_skill.py`
2. Upload to Claude.ai when prompted or through skill management
3. The skill becomes available in your Claude.ai conversations

### Claude Code (CLI)

Install skills locally in the Claude Code environment:

**User-level skills** (available in all projects):
```
~/.claude/skills/skill-name/
```

**Project-level skills** (available in one project):
```
.claude/skills/skill-name/
```

Place the skill directory (not the `.skill` file) in the appropriate location. Claude Code discovers skills automatically.

### GitHub / Version Control

Host skills in a Git repository for team sharing:

```
my-skills-repo/
├── skill-one/
│   ├── SKILL.md
│   ├── scripts/
│   └── references/
├── skill-two/
│   ├── SKILL.md
│   └── assets/
└── README.md
```

Team members clone the repo and symlink or copy skills to their local environment.

### API Integration

For programmatic skill management:
1. Package the skill as a `.skill` file
2. Store in an artifact repository or cloud storage
3. Automate distribution via CI/CD pipelines
4. Version skills alongside application code

### Organization-Wide Deployment

For teams and organizations:

1. **Central repository**: Maintain a shared skills repository
2. **Version control**: Tag skill releases (e.g., `pdf-editor-v1.2`)
3. **Review process**: Require peer review before publishing
4. **Testing pipeline**: Run validation and testing before distribution
5. **Documentation**: Maintain a catalog of available skills with descriptions

## Versioning Best Practices

- Use semantic versioning in your repository tags
- Document breaking changes (description rewording that affects triggering)
- Test skill upgrades in isolation before rolling out
- Keep previous versions available for rollback

## Environment Requirements

If the skill requires specific software or configuration, declare it in the `compatibility` frontmatter field:

```yaml
compatibility: Requires Python 3.8+ with pdfplumber and reportlab packages installed.
```

Keep compatibility notes concise (under 500 characters). Only include genuine requirements - most skills work everywhere without special configuration.
