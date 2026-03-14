---
name: edit-unzipped-docx
description: Edit unzipped docx sources (XML and media), then repack into a valid .docx. Use when modifying an already-unzipped Word document, editing docx XML sources, or producing a docx from an unzipped folder.
---

# Edit unzipped docx and produce a valid docx

## Structure of an unzipped docx

A `.docx` file is a ZIP archive. Unzipped, the root must contain:

- `[Content_Types].xml` — package content types
- `_rels/.rels` — package relationships
- `word/` — main document and resources:
  - `word/document.xml` — **main body text and structure** (primary edit target)
  - `word/styles.xml`, `word/numbering.xml`, `word/settings.xml`, etc.
  - `word/media/`, `word/fonts/` — embedded assets

Do not rename or remove these; repacking must preserve the same layout.

## Finding text in document.xml

- Text lives inside `<w:t>...</w:t>` elements (Word Open XML runs).
- **A single phrase may be split across several `<w:t>` runs** (e.g. one run: "Senior Researcher", next run: " in Quantum Computing"). Grep or search for a unique substring to get exact context.
- Use shell grep to extract runs:
  ```bash
  grep -oE '<w:t[^>]*>[^<]+</w:t>' path/to/word/document.xml | sed 's/<[^>]*>//g'
  ```
- To find all paragraphs and their text content (useful for mapping a document):
  ```python
  import re
  with open('word/document.xml', 'r', encoding='utf-8') as f:
      xml = f.read()
  for m in re.finditer(r'<w:p[ >]', xml):
      s = m.start(); e = xml.find('</w:p>', s) + 6
      texts = re.findall(r'<w:t[^>]*>([^<]+)</w:t>', xml[s:e])
      print(''.join(texts).strip()[:100])
  ```

## Document structure of a two-column CV

Many CVs produced by Word use a single `<w:tbl>` (table) spanning the whole body, where:
- Each `<w:tr>` row = one CV section (e.g. "Permanent positions", "Education")
- The left `<w:tc>` cell holds the section heading (narrow column)
- The right `<w:tc>` cell holds the section content (wide column)

To reorder or remove whole sections, you move or drop `<w:tr>` blocks — this is the safest and most powerful approach for large structural changes.

## Safe XML editing rules

### Rule 1 — Validate after EVERY edit, not just at the end

After each change (text replacement, paragraph removal, run removal), validate immediately:

```python
import xml.etree.ElementTree as ET
try:
    ET.fromstring(xml.encode('utf-8'))   # encode first — avoids encoding-declaration mismatch
except ET.ParseError as e:
    print(f"INVALID XML: {e}")
    # Stop here; do not proceed with more edits
```

Compounding edits on already-corrupted XML produces cascading, hard-to-debug failures.

**Pitfall**: validation only checks well-formedness. A regex that silently matches nothing still produces valid XML — it just leaves the document unchanged. Always confirm the edit took effect (see Rule 8).

### Rule 2 — Use exact tag prefixes; `<w:r` matches `<w:rPr>`

**Critical**: `<w:r` is a prefix of `<w:rPr>`. A search like `rfind('<w:r', 0, idx)` will find `<w:rPr>` if it appears closer to the target text than the actual `<w:r>` opener. Inside a run the structure is:

```xml
<w:r ...>
  <w:rPr>...</w:rPr>   ← closer to <w:t> than <w:r> is
  <w:t>target text</w:t>
</w:r>
```

Always search for `<w:r ` (with a trailing space) or `<w:r>` — never bare `<w:r`:

```python
# WRONG — matches <w:rPr>:
r_start = xml.rfind('<w:r', 0, idx)

# CORRECT — only matches actual run openers:
r_start = max(xml.rfind('<w:r ', 0, idx),
              xml.rfind('<w:r>', 0, idx))
```

The same principle applies to paragraphs (`<w:p ` not `<w:p`) and table cells (`<w:tc ` not `<w:tc`).

### Rule 3 — Use a depth counter for nested block extraction

Do not use regex to find matching closing tags; use a depth-counting parser instead. This is the only correct approach for any element that can be nested.

```python
def find_blocks(xml, open_tag_prefix, close_tag):
    """Return list of (start, end) for each top-level block."""
    blocks = []
    i = 0; depth = 0; start = None; n = len(xml)
    while i < n:
        if xml[i:i+len(open_tag_prefix)] == open_tag_prefix \
                and xml[i+len(open_tag_prefix)] in ' >':
            if depth == 0: start = i
            depth += 1
            i += len(open_tag_prefix)
        elif xml[i:i+len(close_tag)] == close_tag:
            depth -= 1
            if depth == 0 and start is not None:
                blocks.append((start, i + len(close_tag)))
                start = None
            i += len(close_tag)
        else:
            i += 1
    return blocks

# Example — extract all top-level table rows:
trs = find_blocks(xml, '<w:tr', '</w:tr>')
```

### Rule 4 — Reorder/remove large blocks before editing content within them

Apply structural changes (dropping or reordering `<w:tr>` rows) first, validate, then edit content within the remaining rows. Interleaving structural and content changes compounds the risk of corruption.

Workflow:
1. Extract row blocks with the depth-counter parser
2. Reorder / filter to the new order
3. Reassemble: `before + ''.join(selected_rows) + after`
4. **Validate XML**
5. Then do content edits (paragraph removals, text replacements)
6. **Validate XML again after each content edit**

### Rule 5 — Removal helper functions

Use these safe helpers. Never inline the logic.

```python
def remove_para_containing(xml, text):
    """Remove the <w:p>...</w:p> block containing the given text. Returns (xml, found)."""
    idx = xml.find(text)
    if idx == -1:
        return xml, False
    p_start = xml.rfind('<w:p ', 0, idx)
    if p_start == -1:
        p_start = xml.rfind('<w:p>', 0, idx)
    p_end = xml.find('</w:p>', idx) + 6
    return xml[:p_start] + xml[p_end:], True

def remove_run_containing(xml, text):
    """Remove the <w:r>...</w:r> run containing the given text. Returns (xml, found)."""
    idx = xml.find(text)
    if idx == -1:
        return xml, False
    # Use '<w:r ' or '<w:r>' — NOT '<w:r' (would match '<w:rPr>')
    r_start = max(xml.rfind('<w:r ', 0, idx),
                  xml.rfind('<w:r>', 0, idx))
    if r_start == -1:
        return xml, False
    r_end = xml.find('</w:r>', idx) + 6
    return xml[:r_start] + xml[r_end:], True

def remove_hyperlink_containing(xml, text):
    """Remove the <w:hyperlink>...</w:hyperlink> containing the given text. Returns (xml, found)."""
    idx = xml.find(text)
    if idx == -1:
        return xml, False
    h_start = xml.rfind('<w:hyperlink', 0, idx)
    h_end = xml.find('</w:hyperlink>', idx) + len('</w:hyperlink>')
    return xml[:h_start] + xml[h_end:], True
```

After each call, check that `found` is True and validate XML before proceeding.

### Rule 6 — Use a single-pass paragraph filter, not precomputed byte positions

If you need to remove many paragraphs matching different criteria, **do not** precompute a list of byte positions and remove in reverse order. Positions go stale as soon as you make any edit. Instead, use a single-pass builder:

```python
def filter_paras(xml, should_remove):
    """
    Remove paragraphs for which should_remove(text) is True.
    should_remove receives the joined text content of each <w:p> block.
    """
    result_parts = []
    last = 0
    for m in re.finditer(r'<w:p[ >]', xml):
        s = m.start()
        e = xml.find('</w:p>', s) + 6
        texts = re.findall(r'<w:t[^>]*>([^<]+)</w:t>', xml[s:e])
        if not should_remove(''.join(texts)):
            result_parts.append(xml[last:e])
        last = e
    result_parts.append(xml[last:])
    return ''.join(result_parts)
```

### Rule 8 — Confirm that every edit actually changed the XML

`xml.replace(old, new)` and `re.sub(pattern, repl, xml)` are **silent on no-match**: they return the original string unchanged, validation passes, and the file is saved with no effect. Always assert the change occurred:

```python
before = xml
xml = xml.replace(old, new)
assert xml != before, f"Replace had no effect — '{old[:60]}' not found"

# For re.sub, check the count:
xml, n = re.subn(pattern, repl, xml)
assert n > 0, f"Pattern had no match: {pattern}"
```

When a replacement silently fails and the next replacement uses the same anchor assumption, errors compound invisibly across multiple scripts.

### Rule 9 — Inspect run structure before any text edit

Before attempting a text search or replacement, print the run structure of the target paragraph to confirm the text appears as a single continuous string in the XML:

```python
idx = xml.find('distinctive phrase from target paragraph')
assert idx != -1, "Paragraph not found"
p_start = xml.rfind('<w:p ', 0, idx)
p_end   = xml.find('</w:p>', idx) + 6
para = xml[p_start:p_end]
runs = re.findall(r'<w:t[^>]*>([^<]+)</w:t>', para)
print(runs)   # inspect: is your target text in one run, or split?
```

If the target is split across runs (e.g. `['speaks French, Spanish', ', English and Korean']`), searching for `'English and Korean'` will find it, but searching for `'Spanish, English and Korean'` will silently fail. Always anchor your search to a substring that lies **within a single `<w:t>` element**.

### Rule 10 — Use `[^>]*` not `[^/]*` in XML tag patterns

For self-closing XML elements such as `<w:rFonts w:ascii="Cardo"/>`, the correct regex pattern is:

```python
# CORRECT — [^>]* stops at >, never mis-parses the closing />
re.sub(r'<w:rFonts\b[^>]*/?>', '', para)
```

`[^/]*` appears plausible but can stall on attributes that happen to contain `/` (e.g. in URI values), causing the match to fail silently. `>` is the only character guaranteed not to appear inside an unquoted XML attribute value, making `[^>]*` the safe universal choice. The trailing `/?` makes the `>` capture both `>` and `/>` endings explicitly, though `[^>]*` already consumes the `/` in practice.

### Rule 11 — After all edits, check for orphaned tags

Hyperlink and run removals can leave orphaned opening or closing tags. After all edits:

```python
import re
opens  = len(re.findall(r'<w:hyperlink[^/]', xml))
closes = len(re.findall(r'</w:hyperlink>', xml))
if opens != closes:
    print(f"WARNING: {opens} hyperlink opens, {closes} closes")
```

Similarly check `<w:r ` vs `</w:r>` counts if you have removed runs.

## Repacking into a valid .docx

The ZIP must have the **unzipped root contents** at the **archive root** (no extra top-level folder).

From the unzipped folder:

```bash
cd /path/to/UnzippedDocxFolder
zip -r ../Output.docx . -x "*.DS_Store"
```

- Run `zip` from inside the unzipped folder so entries are `[Content_Types].xml`, `_rels/`, `word/`, etc. at the archive root.
- Output to the parent directory (`../Output.docx`).

## Testing the output

After repacking, test with both:

```bash
# 1. Check ZIP integrity
unzip -t Output.docx

# 2. Check Word XML parsability (catches structural errors LibreOffice may silently reject)
python3 -c "
import xml.etree.ElementTree as ET
import zipfile
with zipfile.ZipFile('Output.docx') as z:
    ET.fromstring(z.read('word/document.xml'))
print('XML OK')
"

# 3. Check readable content (catches tag mismatches that break rendering)
pandoc Output.docx -t plain 2>&1 | head -40
```

If pandoc returns `DocxError` or the XML parser raises `ParseError`, the file will also fail in LibreOffice and Google Docs.

## Recommended workflow summary

1. **Map** the document: extract all `<w:tr>` rows and all `<w:p>` paragraphs with their text, printing positions and content. Understand structure before touching anything.
2. **Structural pass**: reorder/drop `<w:tr>` rows using the depth-counter extractor. Validate XML. Run pandoc.
3. **Content pass**: remove paragraphs and runs using the safe helpers above. Validate XML **after each removal**. Assert that each removal changed the XML (Rule 8). Run pandoc after all removals.
4. **Text replacement pass**: before each `xml.replace(old, new)`, inspect the run structure of the target paragraph (Rule 9) to confirm the anchor string is a continuous substring of a single `<w:t>` element. Assert the replacement changed the XML (Rule 8). Use `[^>]*` in any regex patterns over XML attributes (Rule 10). Validate XML after each replacement.
5. **Orphan check**: count `<w:hyperlink>` opens vs closes (Rule 11).
6. **Repack** and run all three tests (unzip -t, ET.fromstring, pandoc).
