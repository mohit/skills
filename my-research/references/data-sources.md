# Data Source Access Patterns

## Email (Gmail via gog)

### Search
```bash
gog gmail search --account <account> "<query>" --limit 50 [--page <token>]
```
- Paginate with `--page` token from output until exhausted
- Query syntax: `from:X`, `to:X`, `before:YYYY/MM/DD`, `after:YYYY/MM/DD`, `label:X`, `is:unread`, `subject:X`
- Combine with OR: `"from:alice OR to:alice"`
- Add `--json` for structured output

### Fetch single email
```bash
gog gmail get <messageId> --account <account> --plain
```
- `--plain` strips HTML, returns text body
- Dump to file for processing: `> /tmp/research-<id>/emails/<messageId>.txt`

### Patterns
- **Person archive**: `"from:person OR to:person"` across all accounts
- **Topic archive**: `"subject:topic OR body:topic"` with date ranges
- **Thread reconstruction**: search returns thread IDs, fetch each message in thread
- **Multi-account**: search each account separately, deduplicate by subject+date

### Gotchas
- OAuth tokens expire — check with `gog auth status --account <account>` before starting
- CHAT-labeled threads are gchat logs, not emails — still fetchable with `gog gmail get`
- Email headers get parsed as links by naive URL extraction (http://gmail.com etc.) — filter these out
- Rate limits: ~50 requests before needing brief pauses

## Obsidian / Markdown Notes

### Access
```bash
# List files
find /path/to/vault -name "*.md" -type f

# Search content
grep -rl "search term" /path/to/vault --include="*.md"

# Read file
cat "/path/to/vault/filename.md"
```

### With obsidian-cli
```bash
obsidian-cli search "query" --vault <vault-name>
obsidian-cli read "note-name" --vault <vault-name>
```

### Patterns
- **Daily notes**: typically `YYYY-MM-DD.md` — iterate by date range
- **Tag search**: `grep -rl "#tagname" /path/to/vault`
- **Folder scope**: limit `find` to specific subdirectory
- **Backlinks**: grep for `[[note-name]]` across vault

## Local Files (documents, papers, code)

### PDF text extraction
```bash
# If pdftotext available
pdftotext file.pdf -

# Or use python
python3 -c "import fitz; doc=fitz.open('file.pdf'); [print(p.get_text()) for p in doc]"
```

### Directory traversal
```bash
find /path/to/folder -type f -name "*.md" -o -name "*.txt" -o -name "*.pdf" | sort
```

### Patterns
- **Academic papers**: extract text, identify title/authors/abstract/sections
- **Code repos**: `find . -name "*.py" -o -name "*.ts"`, read key files, check git log
- **Mixed folders**: list all files, categorize by type, process each appropriately

## Google Calendar

```bash
gog calendar events --account <account> --from <date> --to <date>
```

## Google Contacts

```bash
gog contacts search --account <account> "<name>"
gog contacts list --account <account> --limit 50
```

## Web (supplementary only)

### Search
```
web_search with query parameter
```

### Fetch page content
```
web_fetch with url parameter, extractMode "markdown" or "text"
```

### When to use web
- Verify a claim from the primary source
- Check if a shared link is still alive and what it was about
- Add context (e.g., "who is this person mentioned in the emails?")
- Cross-reference dates or facts
- Fill gaps that primary sources don't cover

### When NOT to use web
- As the primary research source (use the specified corpus instead)
- To pad out thin findings with generic information
- To replace reading the actual source material
