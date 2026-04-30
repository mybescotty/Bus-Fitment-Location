# Writing Large Files — How to Do It Without Failing

## The Problem

When writing large HTML files (300+ lines), the `Write` tool silently stalls or times
out before anything reaches disk. The session appears frozen and nothing is produced.
This happened repeatedly when trying to write `bus-fitment-diagram.html` in one go.

## The Solution — Staged Bash Heredocs

Write the file in 3–4 chunks using `cat` with a heredoc. Each chunk should be
roughly 100–200 lines. Use append mode (`>>`) for every chunk after the first.

### Key Rules

1. Use a **single-quoted** heredoc delimiter — `<< 'EOF'` not `<< EOF`.  
   Single-quoted = no variable expansion, no backtick interpretation. Dollar signs,
   backticks, and special characters inside the HTML/JS are treated as plain text.

2. Choose a **unique delimiter** that will never appear alone on a line inside the
   content — e.g. `STAGE1_EOF`, `BUS_FITMENT_EOF`, `ENDHTML`. Never use plain `EOF`
   for HTML files because comments or script content might accidentally match it.

3. **Echo the line count** after each stage so you can confirm it wrote correctly
   before moving on.

4. Always verify the final file with a line count and a quick syntax check before
   committing.

---

## Template

```bash
# Stage 1 — create the file (head, CSS)
cat > /path/to/file.html << 'STAGE1_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  ...CSS...
</head>
STAGE1_EOF
echo "Stage 1: $(wc -l < /path/to/file.html) lines"

# Stage 2 — append body and HTML structure
cat >> /path/to/file.html << 'STAGE2_EOF'
<body>
  ...HTML...
STAGE2_EOF
echo "Stage 2: $(wc -l < /path/to/file.html) lines"

# Stage 3 — append JavaScript sections 1–4
cat >> /path/to/file.html << 'STAGE3_EOF'
<script>
  /* data, config, CSV parsing, processing */
</script>
STAGE3_EOF
echo "Stage 3: $(wc -l < /path/to/file.html) lines"

# Stage 4 — append JavaScript sections 5–8 and close tags
cat >> /path/to/file.html << 'STAGE4_EOF'
  /* diagram, panel, utilities, init */
</script>
</body>
</html>
STAGE4_EOF
echo "Stage 4: $(wc -l < /path/to/file.html) lines"
```

---

## Sensible Stage Splits for This Project's HTML Structure

| Stage | Content | Approx Lines |
|-------|---------|-------------|
| 1 — Create | `<head>` + all CSS | ~90 |
| 2 — Append | `<body>`, header, upload UI, SVG diagram, HTML panels | ~160 |
| 3 — Append | JS §1–4: zones config, column detection, CSV parsing, data processing | ~150 |
| 4 — Append | JS §5–8: diagram, parts panel, utilities, init + closing tags | ~120 |

---

## What NOT to Do

- **Do not use the `Write` tool for files over ~200 lines** — it stalls silently.
- **Do not use unquoted heredoc** (`<< EOF`) when the content contains `$variables`
  or backticks — they will be interpolated and break the output.
- **Do not write the entire JS as one giant string template** — breaks on backticks
  and `${}` inside the JavaScript.
- **Do not spend time planning the heredoc in text** — just call the Bash tool.
  The failure mode is always "too much thinking, no tool call made".

---

## Quick Verification After Writing

```bash
# Check line count is plausible
wc -l < /path/to/file.html

# Check HTML opens and closes correctly
grep -c "^<html\|^<!DOCTYPE" /path/to/file.html   # expect 1-2
grep -c "</html>" /path/to/file.html               # expect 1

# Check script tags are balanced
grep -c "<script" /path/to/file.html
grep -c "</script>" /path/to/file.html             # counts should match
```

---

## Summary

> Large file = staged Bash heredocs with single-quoted delimiters, ~150 lines per
> stage, echo line count after each. Never use the Write tool for large files.
> Make the tool call immediately — do not narrate the plan first.
