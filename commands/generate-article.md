---
description: Generate blog articles using two-stage pipeline (trend collection + AI generation) and create PR
---

# Generate Article Command

Executes the blog article generation pipeline manually. Uses the **article-generator** agent for full automation.

## Usage

`/generate-article [options]`

## What This Command Does

1. **Check Status** - Show keyword due status, unused topics, and today's articles
2. **Collect Trends** - Run Vertex AI Search for all enabled keywords, extract topics with Claude CLI
3. **Generate Articles** - Create articles from discovered topics using Claude CLI
4. **Create PR** - Export articles to MDX, commit to front submodule, and create GitHub PR

## Prerequisites

- Blog server at `/home/hy-sksem/dev/blog/server`
- Claude CLI installed (`claude --version`)
- Poetry environment set up
- GitHub CLI authenticated (`gh auth status`)
- Vertex AI credentials configured

## Execution

### Full Pipeline (Default)

Run all steps sequentially:

1. **MUST** delegate to **article-generator** agent
2. Agent checks keyword status first
3. Agent runs trend collection (5-15 min per keyword batch)
4. Agent generates articles (3-5 min per article)
5. Agent creates PR with all new articles
6. Agent reports summary table

### Status Check Only

If `--status` is specified:
- Only run Step 1 (check keyword/topic/article status)
- Do NOT run collection, generation, or PR creation
- Display status table and exit

### Skip Collection

If `--skip-collection` is specified:
- Skip Step 2 (trend collection)
- Generate articles from existing unused topics
- Useful when topics were already collected but generation failed

### Specific Keyword

If `--keyword KEYWORD_NAME` is specified:
- Only process the specified keyword
- Run full pipeline for that single keyword

### Reset and Generate

If `--reset-due` is specified:
- Reset `last_generated` to NULL for all enabled keywords before generating
- Ensures all keywords are eligible for generation
- Use when keywords were recently generated but you want to force re-generation

### Create PR Only

If `--pr-only` is specified:
- Skip Steps 1-3
- Only run Step 4 (PR creation)
- Useful when articles are generated but PR creation failed

## Safety Guards

1. **Status First** - Always show status before running pipeline
2. **Continue on Failure** - Individual keyword failures don't stop the pipeline
3. **Quality Check** - Articles are quality-scored before approval
4. **Duplicate Detection** - Checks for duplicate content before saving

## Arguments

$ARGUMENTS:
- `--status` - Only show current status (keywords, topics, articles)
- `--skip-collection` - Skip trend collection, generate from existing topics
- `--keyword NAME` - Process only the specified keyword
- `--reset-due` - Reset last_generated for all keywords before generating
- `--pr-only` - Only create PR (skip collection and generation)
- `--dry-run` - Show what would be done without executing

## Examples

### Full Pipeline
```bash
/generate-article
```

### Check Status
```bash
/generate-article --status
```

### Force Re-generation
```bash
/generate-article --reset-due
```

### Single Keyword
```bash
/generate-article --keyword "クラウドコンピューティング"
```

### Create PR After Manual Generation
```bash
/generate-article --pr-only
```

## Integration

This command invokes the **article-generator** agent which:
- Runs Django management shell commands
- Calls Celery tasks synchronously (not via broker)
- Uses Claude CLI for content generation
- Uses `gh` CLI for PR creation

## Monitoring

Check progress in logs:
```bash
tail -f /home/hy-sksem/dev/blog/server/logs/celery-worker.log
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| All keywords skipped | Use `--reset-due` to reset last_generated |
| Claude CLI returns garbage | Check `cwd='/tmp'` in claude_cli_service.py |
| Topics not discovered | Check Vertex AI credentials and quota |
| PR creation fails | Check git status in `front/` submodule |
| Timeout errors | Increase CLAUDE_CLI_TIMEOUT in .env |
