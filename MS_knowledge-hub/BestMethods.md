# BestMethods — knowledge-hub
Last updated: 2026-06-09

> Hard-won lessons. Read before writing any code. Append as new lessons are learned.

---

## Anthropic Files API — betas must be inside SDK params

The single most important rule in this codebase. **Do not revert this pattern.**

```javascript
// ✅ CORRECT — betas inside call params
import Anthropic, { toFile } from "@anthropic-ai/sdk";

// Upload
await anthropic.beta.files.upload({
  file: await toFile(buffer, filename, { type: "application/pdf" }),
  betas: ["files-api-2025-04-14"],
});

// Ask
await anthropic.beta.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1500,
  messages: [...],
  betas: ["files-api-2025-04-14"],
});

// Delete
await anthropic.beta.files.delete(fileId, {
  betas: ["files-api-2025-04-14"],
});

// ❌ WRONG — causes 404 not_found_error every time
await anthropic.beta.files.upload(
  { file: new File([buffer], name, { type: "application/pdf" }) },
  { headers: { "anthropic-beta": "files-api-2025-04-14" } }
);
```

---

## MicroSynergies logo — always base64 data URI

The SVG logo file is ~16KB of path data. Never inline the SVG paths directly in HTML — the content gets truncated and the logo renders broken.

```html
<!-- ✅ CORRECT -->
<img src="data:image/svg+xml;base64,[base64-encoded-svg]" alt="MicroSynergies" style="height:18px">

<!-- ❌ WRONG — gets truncated at ~16KB -->
<svg xmlns="..." height="18" viewBox="0 0 250 36">
  [thousands of characters of path data]
</svg>
```

To generate the base64 URI:
```python
import base64
svg = open('microsynergies-logo.svg').read().strip()
b64 = base64.b64encode(svg.encode()).decode()
print(f"data:image/svg+xml;base64,{b64}")
```

---

## PDF preferred over DOCX — always recommend PDF to clients

When clients ask which format to use:
- Native Anthropic API support (first-class document type)
- Per-page image parsing included
- Citation support — Claude can reference exact page/section
- Higher accuracy on formatted documents

For DOCX files: convert in Word (File → Save As → PDF) before uploading. DOCX fallback via mammoth works for text-heavy docs but loses images and formatting.

---

## .env setup — skip the cp step

Do not instruct users to `cp .env.example .env`. Instruct them to create `.env` directly with any text editor:

```
ANTHROPIC_API_KEY=sk-ant-...
PORT=3000
```

---

## Temporary file cleanup — always in a finally block

Always delete temp files from `uploads/` immediately after reading.

```javascript
const tempPath = req.file.path;
try {
  const buffer = fs.readFileSync(tempPath);
  // ... do work ...
} finally {
  if (fs.existsSync(tempPath)) fs.unlinkSync(tempPath);
}
```

---

## Wiki — content structure before code

Don't scaffold the framework before auditing the content. The wiki structure should be driven by what's actually in the FAQ doc, not a generic template. Steps:

1. Read Brad's document thoroughly
2. Identify all topic areas and natural groupings
3. Sketch the sidebar navigation
4. Then scaffold the framework

---

## Wiki — Dale's edit workflow matters more than the stack

Dale is not necessarily technical. The editing experience for Dale is a primary UX concern. Options to discuss at kickoff:

- **GitHub web editor** — no local tools needed, but requires comfort with Git concepts
- **Netlify CMS / Decap CMS** — Git-backed with a web admin panel (no GitHub knowledge needed)
- **Notion as CMS** — Dale may already be comfortable with Notion

Choose the editing experience first, then pick the stack that supports it.

---

## Wiki — keep wiki-content/ and docs/ strictly separate

- `wiki-content/` — the actual wiki pages (markdown files that become the published site)
- `docs/` — project management files (SessionStarter, Roadmap, etc. — never published)

Do not let project management files leak into the published wiki.

---

## OOXML border fix — docx Node.js library post-processing

The `docx` Node.js library generates `w:tcBorders` with wrong element names and order for the newer OOXML schema. Fix by post-processing the generated ZIP with Python after `Packer.toBuffer()`:

```python
import zipfile, re, os

def fix_docx_borders(src_path):
    tmp_path = src_path + ".tmp"

    with zipfile.ZipFile(src_path, 'r') as zin:
        with zipfile.ZipFile(tmp_path, 'w', zipfile.ZIP_DEFLATED) as zout:
            for item in zin.infolist():
                data = zin.read(item.filename)
                if item.filename == 'word/document.xml':
                    content = data.decode('utf-8')

                    def fix_border_block(m, container):
                        b = m.group(0)
                        b = re.sub(r'<w:left\b', '<w:start', b)
                        b = re.sub(r'<w:right\b', '<w:end', b)
                        def get(t):
                            x = re.search(rf'<w:{t}\b[^>]*/>', b)
                            return x.group(0) if x else ''
                        order = ['top', 'start', 'bottom', 'end', 'insideH', 'insideV']
                        return f'<w:{container}>' + ''.join(get(t) for t in order) + f'</w:{container}>'

                    for c in ['tcBorders', 'tblBorders']:
                        content = re.sub(rf'<w:{c}>.*?</w:{c}>',
                            lambda m, c=c: fix_border_block(m, c), content, flags=re.DOTALL)

                    for c in ['tcMar', 'tblMar']:
                        content = re.sub(rf'<w:{c}>.*?</w:{c}>',
                            lambda m: m.group(0).replace('<w:left ', '<w:start ').replace('<w:right ', '<w:end '),
                            content, flags=re.DOTALL)

                    # pBdr: KEEP left/right — paragraph borders use old names
                    data = content.encode('utf-8')
                zout.writestr(item, data)

    os.replace(tmp_path, src_path)
```

| Container | Names | Reorder required |
|-----------|-------|-----------------|
| `w:tcBorders` | `start` / `end` | Yes — `top, start, bottom, end` |
| `w:tblBorders` | `start` / `end` | Yes — `top, start, bottom, end` |
| `w:tcMar` | `start` / `end` | No |
| `w:tblMar` | `start` / `end` | No |
| `w:pBdr` | `left` / `right` (keep) | No |
