# Phase: Quarto Report Rendering (Replace Gotenberg)

**Status:** Planning
**Created:** 2026-02-08
**Priority:** Next (can be done independently of anaerobic design tools)

---

## Objective

Replace the Gotenberg HTML-to-PDF conversion with Quarto rendering directly on the QSDsan VM. This eliminates the Gotenberg Docker container dependency, unifies the two separate report generation paths, and produces higher-quality reports.

---

## Current State: Two Disconnected Report Paths

### Path 1: Engine QMD Generation (Existing, Not Rendered)

The engine already generates professional QMD reports:

```
Simulation Results
  → normalize_results_for_report()       (reports/qmd_builder.py)
  → Jinja2 template rendering            (reports/templates/*.qmd)
  → QMD file + CSS + plot PNGs saved     (jobs/flowsheets/{session_id}/)
  → STOPS HERE (never rendered to HTML/PDF)
```

**Files involved:**
- `reports/qmd_builder.py` - Normalization, data prep, Jinja2 rendering (~900 lines)
- `reports/templates/aerobic_report.qmd` - Aerobic MBR template (324 lines)
- `reports/templates/anaerobic_report.qmd` - Anaerobic CSTR template (313 lines)
- `reports/templates/report.css` - Professional A4-optimized stylesheet (558 lines)

**QMD template features:**
- YAML frontmatter with `format: html`, `css: report.css`, `embed-resources: true`
- KPI grid cards (COD removal, TN removal, TP removal, SRT)
- Diagnostic panels with status indicators (green/yellow/red)
- Influent/effluent comparison tables
- Flowsheet SVG diagrams (conditional)
- Per-unit analysis tables
- Time-series plot references (convergence, nutrients, biogas)
- Process-specific sections (nitrogen, phosphorus, biomass, etc.)
- Print-optimized CSS (A4 page breaks, responsive layout)

### Path 2: n8n HTML Generation (Current Production)

The n8n workflow generates a **separate, simpler** HTML report:

```
Simulation Results (from API)
  → "Generate Report" Code node    (JavaScript, builds HTML string)
  → "Create HTML File" node        (converts string to binary)
  → "Convert to PDF" node          (Gotenberg: HTML → PDF)
  → "Upload PDF to Supabase"       (cloud storage)
```

**Problems with this approach:**
- Duplicates report logic (JS in n8n vs Python in engine)
- Simpler formatting than the QMD templates
- No KPI cards, diagnostic panels, or advanced styling
- Requires Gotenberg Docker container (extra service to maintain)
- HTML-to-PDF conversion can timeout (120s limit)
- Cannot embed Python-generated plots

---

## Proposed Architecture

### New MCP Tool: `render_report`

Add a new tool that takes a QMD file path (or session_id) and renders it to HTML and/or PDF using Quarto installed on the VM.

```python
@mcp.tool()
async def render_report(
    session_id: str = None,
    job_id: str = None,
    qmd_path: str = None,
    output_format: str = "html",         # "html", "pdf", or "both"
    embed_resources: bool = True,        # Inline CSS/images for standalone HTML
) -> Dict[str, Any]:
    """
    Render a QMD report to HTML and/or PDF using Quarto.

    Provide either session_id (flowsheet), job_id (direct simulation),
    or qmd_path (custom QMD file).

    Returns paths to rendered files.
    """
```

**Returns:**
```json
{
  "status": "success",
  "html_path": "jobs/flowsheets/abc123/report.html",
  "pdf_path": "jobs/flowsheets/abc123/report.pdf",
  "html_size_kb": 245,
  "pdf_size_kb": 380,
  "render_time_seconds": 3.2
}
```

### Updated Pipeline

```
Simulation Results
  → normalize_results_for_report()       (existing)
  → Jinja2 template rendering            (existing)
  → QMD + CSS + plots saved              (existing)
  → NEW: Quarto render (QMD → HTML/PDF)
  → NEW: get_artifact returns HTML/PDF
```

### n8n Integration (Simplified)

The n8n workflow can replace three nodes with one API call:

**Before (3 nodes):**
```
Generate Report (JS) → Create HTML File → Convert to PDF (Gotenberg)
```

**After (1 node):**
```
HTTP Request: POST /api/render_report
  Body: {"session_id": "abc123", "output_format": "pdf"}
  Response: {"pdf_path": "...", "status": "success"}

Then: GET /api/get_artifact?path=...
  Response: PDF binary (or signed URL in cloud mode)
```

---

## Implementation Plan

### Step 1: Install Quarto on the VM

SSH into the QSDsan VM and install Quarto:

```bash
# Download Quarto
wget https://github.com/quarto-dev/quarto-cli/releases/download/v1.6.42/quarto-1.6.42-linux-amd64.deb

# Install
sudo dpkg -i quarto-1.6.42-linux-amd64.deb

# For PDF output, install TinyTeX
quarto install tinytex

# Verify
quarto --version
quarto check
```

**Note:** The QMD templates use `engine: markdown` in frontmatter (no Python code chunks), so **Jupyter is NOT required** on the VM. Quarto only needs to process markdown + CSS.

### Step 2: Update QMD Templates for Quarto Compatibility

The existing QMD templates use Jinja2 syntax (`{{ data.value }}`). Since Jinja2 rendering happens **before** Quarto, the QMD files that Quarto receives are already fully rendered markdown. Verify this works correctly:

**Check for conflicts:**
- Jinja2 uses `{{ }}` and `{% %}` - these should be fully resolved before Quarto sees the file
- Quarto uses `` `{{< ... >}}` `` for shortcodes - no conflicts expected
- YAML frontmatter must be valid after Jinja2 rendering

**Template adjustments needed:**

1. Ensure `format: html` in YAML frontmatter (already present)
2. Add PDF format option:
   ```yaml
   format:
     html:
       toc: true
       toc-depth: 2
       css: report.css
       embed-resources: true
     pdf:
       toc: true
       toc-depth: 2
       documentclass: article
       geometry: margin=2.5cm
   ```
3. Verify `report.css` is copied alongside QMD file (already handled in `qmd_builder.py`)

### Step 3: Add `render_report` Function to Core

```python
# core/report_renderer.py (NEW)

import subprocess
import shutil
from pathlib import Path
from typing import Dict, Any, Optional
import logging

logger = logging.getLogger(__name__)


def find_quarto() -> Optional[Path]:
    """Find the Quarto executable."""
    quarto = shutil.which("quarto")
    if quarto:
        return Path(quarto)
    # Check common install locations
    for path in [
        Path("/usr/local/bin/quarto"),
        Path("/usr/bin/quarto"),
        Path("C:/Users/gaierr/AppData/Local/Programs/Quarto/bin/quarto.exe"),
    ]:
        if path.exists():
            return path
    return None


def render_qmd(
    qmd_path: Path,
    output_format: str = "html",
    timeout_seconds: int = 60,
) -> Dict[str, Any]:
    """
    Render a QMD file to HTML and/or PDF using Quarto.

    Args:
        qmd_path: Path to the QMD file
        output_format: "html", "pdf", or "both"
        timeout_seconds: Render timeout

    Returns:
        Dict with paths to rendered files and status
    """
    quarto = find_quarto()
    if not quarto:
        return {
            "status": "error",
            "error": "Quarto not found. Install from https://quarto.org",
        }

    if not qmd_path.exists():
        return {
            "status": "error",
            "error": f"QMD file not found: {qmd_path}",
        }

    results = {}
    formats = ["html", "pdf"] if output_format == "both" else [output_format]

    for fmt in formats:
        try:
            cmd = [
                str(quarto),
                "render",
                str(qmd_path),
                "--to", fmt,
                "--no-browser",
            ]

            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                timeout=timeout_seconds,
                cwd=str(qmd_path.parent),
            )

            output_path = qmd_path.with_suffix(f".{fmt}")

            if result.returncode == 0 and output_path.exists():
                results[f"{fmt}_path"] = str(output_path)
                results[f"{fmt}_size_kb"] = round(output_path.stat().st_size / 1024, 1)
            else:
                results[f"{fmt}_error"] = result.stderr or "Render failed"

        except subprocess.TimeoutExpired:
            results[f"{fmt}_error"] = f"Render timed out ({timeout_seconds}s)"
        except Exception as e:
            results[f"{fmt}_error"] = str(e)

    results["status"] = "success" if any(
        k.endswith("_path") for k in results
    ) else "error"

    return results
```

### Step 4: Add MCP Tool to `server.py`

```python
@mcp.tool()
async def render_report(
    session_id: str = None,
    job_id: str = None,
    qmd_path: str = None,
    output_format: str = "html",
    timeout_seconds: int = 60,
) -> Dict[str, Any]:
    """
    Render a QMD report to HTML and/or PDF using Quarto.

    Provide session_id for flowsheet reports, job_id for direct
    simulation reports, or qmd_path for a custom QMD file.

    Args:
        session_id: Flowsheet session ID
        job_id: Direct simulation job ID
        qmd_path: Direct path to QMD file
        output_format: "html", "pdf", or "both"
        timeout_seconds: Render timeout (default 60s)

    Returns:
        Dict with rendered file paths and status
    """
```

### Step 5: Add CLI Command

```bash
python cli.py render-report \
  --session abc123 \
  --format pdf \
  --timeout 60
```

### Step 6: Update Docker Image

Add Quarto to the Dockerfile:

```dockerfile
# Install Quarto
RUN wget https://github.com/quarto-dev/quarto-cli/releases/download/v1.6.42/quarto-1.6.42-linux-amd64.deb \
    && dpkg -i quarto-1.6.42-linux-amd64.deb \
    && rm quarto-1.6.42-linux-amd64.deb

# Install TinyTeX for PDF output (optional, adds ~300MB)
RUN quarto install tinytex --no-prompt
```

**Docker image size impact:**
- Quarto CLI: ~200MB
- TinyTeX (for PDF): ~300MB
- Total increase: ~500MB (current image likely ~1-2GB with QSDsan)

**Alternative:** Skip TinyTeX in Docker, render HTML only in production (HTML is self-contained with `embed-resources: true`), and let users render PDF locally with Quarto.

### Step 7: Update n8n Workflow

Replace the three report nodes with a simpler flow:

**Option A: Engine renders report (recommended)**

The simulation already generates QMD via `--report` flag. Add a render step:

```
Submit Simulation → Poll Status → Get Results
  → HTTP Request: POST /api/render_report
      Body: {"job_id": "xyz", "output_format": "html"}
  → HTTP Request: GET /api/get_artifact?job_id=xyz&filename=report.html
  → Upload to Supabase
```

This eliminates:
- "Generate Report" code node (JavaScript)
- "Create HTML File" node
- "Convert to PDF" node (Gotenberg)

**Option B: Engine returns rendered HTML in results**

Add `render_report=true` parameter to `simulate_system`:
```
POST /api/simulate_system
Body: {..., "report": true, "render_report": "html"}
```

Results include the rendered HTML path, which can be fetched via `get_artifact`.

### Step 8: Retire Gotenberg (After Validation)

Once Quarto rendering is verified in production:

1. Remove Gotenberg Docker container from VM
2. Remove Gotenberg URL from n8n Env Parameters
3. Free port 3000 on the VM

---

## CSS Considerations

### Existing CSS

The engine already has `reports/templates/report.css` (558 lines) with:
- Professional grayscale color scheme
- IBM Plex Mono/Sans fonts
- KPI card grid (4-column)
- Status indicator dots (green/yellow/red)
- Diagnostic panels
- Print-optimized layout (A4 page breaks)
- Responsive design (<768px)
- Right-aligned numeric tables with striped rows

### CSS for Quarto

Quarto supports custom CSS via the YAML frontmatter:
```yaml
format:
  html:
    css: report.css
    embed-resources: true
```

The existing `report.css` is already designed for this. **No new CSS needed.**

For PDF output, Quarto uses a different rendering path (LaTeX or wkhtmltopdf). CSS applies to HTML; PDF styling uses:
```yaml
format:
  pdf:
    documentclass: article
    geometry: margin=2.5cm
    fontsize: 11pt
    colorlinks: true
```

### CSS Handling in Pipeline

The `qmd_builder.py` already copies `report.css` alongside the QMD file:
```python
# In generate_report()
css_src = TEMPLATE_DIR / "report.css"
css_dst = output_dir / "report.css"
shutil.copy2(css_src, css_dst)
```

This means the CSS is always available for Quarto rendering. **No changes needed.**

---

## Comparison: Gotenberg vs Quarto

| Aspect | Gotenberg (Current) | Quarto (Proposed) |
|--------|--------------------|--------------------|
| Service | Separate Docker container | Installed on VM (or in Docker) |
| Port | 3000 | None (CLI tool) |
| Input | HTML string | QMD file (already generated) |
| Output | PDF only | HTML, PDF, DOCX, EPUB |
| Styling | Inline CSS in HTML | External CSS + Quarto themes |
| Charts | Must be pre-rendered as images | Can execute Python code chunks |
| Table of Contents | Manual HTML | Automatic (`toc: true`) |
| Cross-references | Not supported | Native (`@fig-polar`, `@tbl-results`) |
| Math | Limited | Full LaTeX support |
| Report quality | Basic (Chromium render) | Professional (LaTeX/Pandoc) |
| Timeout issues | Yes (120s limit) | Rarely (renders locally) |
| Dependency | Docker container running | Quarto CLI installed |
| Maintenance | Container restart on VM reboot | None after install |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Quarto not available in Docker | Low | Medium | Quarto has official Docker images; or use HTML-only mode |
| PDF rendering slow for complex reports | Low | Low | Set timeout; HTML is instant and sufficient for most uses |
| TinyTeX install adds Docker size | Medium | Low | Make PDF optional; HTML with embed-resources is self-contained |
| Jinja2 `{{ }}` conflicts with Quarto | Low | High | Jinja2 renders first; Quarto sees plain markdown. Test thoroughly |
| n8n workflow migration effort | Low | Low | Can run both paths in parallel during transition |

---

## Acceptance Criteria

- [ ] `quarto --version` works on the VM
- [ ] `render_report` MCP tool returns HTML for a known session
- [ ] `render_report` MCP tool returns PDF for a known session
- [ ] Rendered HTML matches quality of existing QMD templates
- [ ] CSS styling renders correctly (KPI cards, tables, status indicators)
- [ ] Flowsheet SVG diagrams embed correctly
- [ ] Plot PNGs render correctly in output
- [ ] `render-report` CLI command works
- [ ] n8n workflow can call `/api/render_report` and upload result
- [ ] Docker image builds with Quarto included
- [ ] Gotenberg can be removed after validation
- [ ] AI analysis section renders when `ai_analysis` is provided
- [ ] AI analysis section is hidden when `ai_analysis` is absent
- [ ] n8n can pass AI analysis text into `render_report` API call
- [ ] Existing 450+ tests still pass

---

## Dependencies

- Quarto CLI (install on VM and in Docker)
- TinyTeX (optional, for PDF output only)
- No new Python packages (subprocess call to Quarto CLI)
- Existing `reports/qmd_builder.py` (generates QMD)
- Existing `reports/templates/report.css` (styling)

---

## Estimated Effort

| Task | Estimate |
|------|----------|
| Install Quarto on VM | Quick |
| `core/report_renderer.py` | Small module (~80 lines) |
| `server.py` MCP tool wrapper | Thin wrapper |
| `cli.py` render-report command | Thin wrapper |
| Template YAML adjustments | Minor (add PDF format option) |
| Tests | ~10 tests |
| Docker image update | Dockerfile change + rebuild |
| n8n workflow update | Replace 3 nodes with 1 HTTP call |
| Gotenberg retirement | Remove container after validation |

---

## HTML Delivery to End Users

### Why Self-Contained HTML Works

Quarto's `embed-resources: true` setting produces a **single .html file** with all CSS, images (base64-encoded), and fonts inlined. No external dependencies. The user's **browser is the renderer** - they click a link, the browser displays the report. No server-side rendering required.

### Recommended Approach: Supabase Storage (Option A)

Upload the self-contained HTML to Supabase exactly like PDFs are uploaded today. Supabase serves files with public URLs that render directly in the browser.

**Delivery URL format:**
```
https://egrzvwnjrtpwwmqzimff.supabase.co/storage/v1/object/public/panicleDevelop_1/{session_id}/{analysis_type}/{session_id}-report.html
```

Users click the link → browser downloads and renders the HTML instantly.

**Why this works:**
- Supabase storage serves static files with correct MIME types (`text/html`)
- Self-contained HTML has zero external dependencies
- Works on any device with a browser (desktop, mobile, tablet)
- No server-side rendering infrastructure needed
- Already have Supabase upload logic in n8n workflow

**Comparison with PDF:**

| Aspect | Self-Contained HTML | PDF |
|--------|--------------------|----|
| File size | ~250-500 KB | ~300-800 KB |
| Rendering | Browser (instant) | PDF viewer |
| Interactivity | Links, TOC navigation, hover effects | Static |
| Mobile experience | Responsive (CSS media queries) | Pinch-to-zoom |
| Copy/paste | Full text selection | Often garbled |
| Printing | Browser print (Ctrl+P) | Direct |
| Search engines | Indexable | Not indexable |
| Offline | Works after download | Works after download |

**Recommendation:** Upload **both** HTML and PDF. HTML as the primary link for viewing, PDF for downloading/archiving. The HTML is the richer experience; the PDF is the universal fallback.

### Alternative Options Considered

**Option B: Google Cloud Storage (GCS) Signed URLs**

Works identically to Supabase. Engine already supports GCS in cloud mode (Phase ENV-1). Signed URLs serve HTML with correct content type. Good if migrating away from Supabase.

**Option C: Static File Server on VM (nginx)**

Add nginx on the VM to serve reports at `http://{static_ip}:8082/reports/{session_id}/report.html`. More infrastructure to maintain. Not recommended unless Supabase is unavailable.

### n8n Workflow Changes for HTML Delivery

**Current flow (PDF via Gotenberg):**
```
Generate Report (JS) → Create HTML File → Gotenberg → Upload PDF → Return PDF URL
```

**New flow (HTML via Quarto):**
```
POST /api/render_report → GET /api/get_artifact → Upload HTML to Supabase → Return HTML URL
```

**Dual format flow (recommended):**
```
POST /api/render_report (format: "both")
  → GET /api/get_artifact (report.html) → Upload HTML to Supabase
  → GET /api/get_artifact (report.pdf)  → Upload PDF to Supabase
  → Return both URLs to user
```

The n8n "Final Response" node can provide:
```json
{
  "report_html_url": "https://...supabase.co/.../report.html",
  "report_pdf_url": "https://...supabase.co/.../report.pdf",
  "message": "View report: [HTML] | Download: [PDF]"
}
```

---

## AI Expert Analysis Integration

### Background

The n8n workflow includes an AI analysis step that prompts an LLM for an expert opinion on the simulation results and the user's original query. This produces a markdown text object that is currently:

1. Merged with simulation results in n8n (JavaScript)
2. Uploaded separately to Supabase as `{session_id}-AI_Analysis.md`
3. **NOT included** in the engine's QMD report templates

This means the final report and the AI opinion are delivered as two separate artifacts, requiring users to cross-reference them manually.

### Proposed Solution: Merge AI Analysis into QMD Templates

Add an optional `ai_analysis` section to both QMD templates. Since the AI analysis content is already markdown, it renders natively in Quarto without any conversion.

#### Template Changes

Add to both `aerobic_report.qmd` and `anaerobic_report.qmd` (before the footer):

```markdown
{% if data.ai_analysis %}
## AI Expert Analysis

{{ data.ai_analysis }}

{% if data.ai_analysis_meta %}
::: {.callout-note appearance="minimal"}
**Analysis by:** {{ data.ai_analysis_meta.model | default('AI') }} |
**Generated:** {{ data.ai_analysis_meta.timestamp | default(meta.report_date) }}
:::
{% endif %}
{% endif %}
```

#### Data Flow Changes

**`reports/qmd_builder.py`** - Both `_prepare_aerobic_data()` and `_prepare_anaerobic_data()` pass through the `ai_analysis` field:

```python
return {
    ...existing fields...
    'ai_analysis': data.get('ai_analysis'),          # Markdown string
    'ai_analysis_meta': data.get('ai_analysis_meta'), # Optional metadata
}
```

**`render_report` MCP tool** - Accept optional `ai_analysis` parameter:

```python
@mcp.tool()
async def render_report(
    session_id: str = None,
    job_id: str = None,
    qmd_path: str = None,
    output_format: str = "html",
    ai_analysis: str = None,           # NEW: Markdown text from AI opinion
    ai_analysis_model: str = None,     # NEW: Which AI model was used
    timeout_seconds: int = 60,
) -> Dict[str, Any]:
```

**`render-report` CLI command** - Accept optional `--ai-analysis-file` parameter:

```bash
python cli.py render-report \
  --session abc123 \
  --format html \
  --ai-analysis-file path/to/ai_analysis.md
```

#### n8n Integration (Unified)

**Current flow (separate artifacts):**
```
Simulation Results → Generate Report (JS) → Upload Report PDF
                   → Submit AI Analysis   → Upload AI_Analysis.md
```

**New flow (single unified report):**
```
Simulation Results → Submit AI Analysis → Get AI Opinion
  → POST /api/render_report
      Body: {
        "session_id": "abc123",
        "output_format": "both",
        "ai_analysis": "## Key Findings\n\nThe simulation shows..."
      }
  → GET /api/get_artifact (report.html)  → Upload to Supabase
  → GET /api/get_artifact (report.pdf)   → Upload to Supabase
```

The AI analysis and simulation report become a **single document**, eliminating the need for separate uploads and giving users one cohesive deliverable.

#### Styling

The AI analysis section will use the existing `report.css` styles. Additionally, add a subtle visual separator:

```css
/* AI Analysis section styling */
.ai-analysis-section {
    border-top: 2px solid #e0e0e0;
    margin-top: 2rem;
    padding-top: 1.5rem;
}

.ai-analysis-section h2::before {
    content: "";
    display: inline-block;
    width: 4px;
    height: 1.2em;
    background: #4a90d9;
    margin-right: 0.5em;
    vertical-align: text-bottom;
}
```

#### Backward Compatibility

- If `ai_analysis` is `None` or empty, the section is completely hidden (Jinja2 `{% if %}` guard)
- Existing reports without AI analysis continue to work unchanged
- The AI analysis is optional in all interfaces (MCP, CLI, n8n)
- `normalize_results_for_report()` preserves `ai_analysis` if present in the results dict

---

## Future Extensions

1. **DOCX output** - Quarto can render to Word documents for clients who prefer it
2. **Slide decks** - Quarto supports RevealJS presentations from the same QMD
3. **Interactive HTML** - With `embed-resources: false`, reports can include interactive Plotly charts
4. **Report themes** - Quarto supports custom themes for branded reports
5. **Multi-language** - Quarto supports i18n for report headers/labels
