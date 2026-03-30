# VistA Documentation Library — Web Table Publishing Guide

## Overview

The VistA Documentation Library inventory is published as an interactive, browser-based
table at `https://vistadocs.github.io`. Users can browse, sort, and filter 8,800+ VistA
documents by section, package, doc type, status, format, and layer — with direct links to
each document on the VA VDL site.

No server, no database, no build step. Everything runs in the browser from two static files.

---

## Strategy

### Goals
- Browse the full VDL inventory without downloading files
- Filter by multiple dimensions simultaneously
- Click through to the source document on the VA VDL
- Zero hosting cost, zero maintenance overhead
- Easy to update: drop in a new CSV and push

### Approach: Pure Static Site

The site is two files:

| File | Purpose |
|---|---|
| `index.html` | Complete application — layout, styles, logic |
| `vdl_inventory_enriched.csv` | Data source — the enriched VDL inventory |

Hosted on **GitHub Pages** from the `vistadocs/vistadocs.github.io` repository.
GitHub Pages serves any static file in the repo at `https://vistadocs.github.io`.

---

## Tooling

All libraries are loaded from CDN — nothing is installed locally.

| Library | Version | Purpose |
|---|---|---|
| [PapaParse](https://www.papaparse.com/) | 5.4.1 | Parse the CSV file in the browser |
| [jQuery](https://jquery.com/) | 3.7.1 | Required by DataTables |
| [DataTables](https://datatables.net/) | 2.0.8 | Interactive table: sort, search, paginate |
| [DataTables Buttons](https://datatables.net/extensions/buttons/) | 3.0.2 | Column visibility toggle button |
| [DataTables ColVis](https://datatables.net/extensions/buttons/examples/column_visibility/) | 3.0.2 | Show/hide columns UI |

**Why DataTables?**
DataTables is the most widely used JavaScript table library (15+ years, massive community).
It handles sorting, pagination, global search, and per-column search out of the box, with
no build step and reliable CDN delivery.

**Why PapaParse?**
Browsers block `fetch()` on `file://` URLs (CORS), so the CSV cannot be loaded by a plain
`<script>`. PapaParse handles streaming CSV download, header detection, type coercion, and
empty-line skipping in one call.

---

## Data Source

**File:** `vdl_inventory_enriched.csv`

The CSV is the output of the VistA Docs pipeline `enrich` stage. It contains one row per
VDL document entry (8,834 rows) with fields crawled from the VA VDL and enriched with
derived metadata.

Fields used by the web table:

| CSV Field | Column | Notes |
|---|---|---|
| `section_name` | Section | Clinical, Infrastructure, etc. |
| `app_name_abbrev` | Package | Short package code, e.g. ADT, PSO |
| `doc_label` / `doc_code` | Doc Type | Human-readable doc type label |
| `app_status` | Status | active, archive, decommissioned |
| `doc_format` | Format | pdf, docx, etc. |
| `doc_layer` | Layer | patch, anchor, etc. |
| `doc_title` | Document Title | Linked to `doc_url` |
| `app_name_full` | Full Name | Hidden by default |
| `patch_id` | Patch | Hidden by default |
| `doc_url` | — | Used to hyperlink the Document Title |

**To update the data:** replace `vdl_inventory_enriched.csv` in the repo with a new export
from the pipeline and push. No changes to `index.html` are needed unless CSV field names change.

---

## Implementation

### HTML Structure

```html
<table id="vdl" class="display">
  <thead>
    <tr>                        <!-- Column headers (row 1) -->
      <th>Section</th>
      <th>Package</th>
      ...
    </tr>
    <tr class="filter-row">     <!-- Per-column filter controls (row 2) -->
      <th></th><th></th>...
    </tr>
  </thead>
  <tbody></tbody>               <!-- DataTables populates this from the CSV data -->
</table>
```

The `<tbody>` is left empty in HTML. DataTables receives the parsed CSV rows as a JavaScript
array and renders all rows itself.

### Loading and Parsing the CSV

```javascript
Papa.parse('vdl_inventory_enriched.csv', {
  download: true,       // fetch the file by URL
  header: true,         // use first row as field names
  skipEmptyLines: true,
  complete: function(results) {
    var rows = results.data;  // array of plain objects, one per CSV row
    // ... initialize DataTables with rows
  }
});
```

### Column Definitions

Each column is declared in a `COLS` array with three properties:

```javascript
var COLS = [
  { title: 'Section',        field: 'section_name',    type: 'select' },
  { title: 'Package',        field: 'app_name_abbrev', type: 'select' },
  { title: 'Doc Type',       field: 'doc_label',       type: 'select' },
  { title: 'Status',         field: 'app_status',      type: 'select' },
  { title: 'Format',         field: 'doc_format',      type: 'select' },
  { title: 'Layer',          field: 'doc_layer',       type: 'select' },
  { title: 'Document Title', field: 'doc_title',       type: 'none'   },
  { title: 'Full Name',      field: 'app_name_full',   type: 'none'   },
  { title: 'Patch',          field: 'patch_id',        type: 'none'   },
];
```

- `type: 'select'` — column gets a dropdown filter populated from its unique values
- `type: 'none'` — column has no filter control (high-cardinality fields like title and patch)

### Orthogonal Data Rendering

DataTables supports **orthogonal data**: a column can return different values depending on
whether DataTables needs it for `display`, `filter`, or `sort`. This is critical for columns
that render HTML badges or hyperlinks — the filter must search plain text, not HTML markup.

```javascript
render: function(row, type) {
  var raw = row[col.field] || '';

  // Plain text for filtering and sorting
  if (type !== 'display') return raw;

  // HTML for display only
  if (col.field === 'app_status') {
    var cls = { active: 'badge-active', archive: 'badge-archive',
                decommissioned: 'badge-decommissioned' }[raw] || '';
    return '<span class="badge ' + cls + '">' + esc(raw) + '</span>';
  }
  if (col.field === 'doc_title') {
    return row.doc_url
      ? '<a href="' + esc(row.doc_url) + '" target="_blank">' + esc(raw) + '</a>'
      : esc(raw);
  }
  return esc(raw);
}
```

Without orthogonal rendering, filtering the Status column for "active" would fail because
the cell contains `<span class="badge badge-active">active</span>` and the regex `^active$`
would not match.

### Per-Column Dropdown Filters

Unique values for each `select` column are computed from the raw CSV data before DataTables
initializes — not from rendered cell HTML:

```javascript
var uniqueVals = {};
COLS.forEach(function(col) {
  if (col.type !== 'select') return;
  var seen = {};
  rows.forEach(function(r) { if (r[col.field]) seen[r[col.field]] = true; });
  uniqueVals[col.field] = Object.keys(seen).sort();
});
```

The `<select>` elements are injected into the `filter-row` cells inside `initComplete`,
after DataTables has fully initialized:

```javascript
initComplete: function() {
  var api = this.api();
  COLS.forEach(function(col, i) {
    var tf = $($('#vdl thead tr.filter-row th').get(i));
    if (col.type === 'select') {
      var sel = $('<select><option value="">All</option></select>');
      uniqueVals[col.field].forEach(function(v) {
        sel.append($('<option>').val(v).text(v));
      });
      sel.appendTo(tf).on('change', function() {
        var val = $(this).val();
        api.column(i).search(
          val ? '^' + $.fn.dataTable.util.escapeRegex(val) + '$' : '',
          true,   // regex
          false   // no smart search
        ).draw();
      });
    }
  });
}
```

The regex is anchored (`^value$`) so selecting "patch" does not match "patch-123".

### Filter Row / Column Visibility Sync

When the user hides a column via the **Show/hide columns** button, the corresponding
filter cell in `filter-row` must also hide. This is handled by listening to DataTables'
`column-visibility` event:

```javascript
function syncFilterRow() {
  api.columns().every(function(i) {
    var th = $($('#vdl thead tr.filter-row th').get(i));
    this.visible() ? th.show() : th.hide();
  });
}
syncFilterRow();                          // run once on load
api.on('column-visibility.dt', syncFilterRow);  // run on every toggle
```

### DataTables Configuration Summary

```javascript
$('#vdl').DataTable({
  data: rows,             // raw JS objects from PapaParse
  columns: dtColumns,     // column config with orthogonal render functions
  pageLength: 50,
  lengthMenu: [25, 50, 100, 250],
  order: [[1, 'asc'], [6, 'asc']],   // sort by Package then Document Title
  dom: 'Blfrtip',         // layout: Buttons, length, filter, table, info, pagination
  buttons: [{ extend: 'colvis', text: 'Show/hide columns' }],
  columnDefs: [{ targets: [7, 8], visible: false }],  // hide Full Name, Patch by default
  orderCellsTop: true,    // sort clicks go to first header row, not filter row
});
```

---

## Local Development

Test locally before pushing. Python's built-in HTTP server replicates the GitHub Pages
environment exactly — in particular, it serves files over HTTP so `fetch()` works:

```bash
cd ~/data/vista-docs/www
python3 -m http.server 8000
# browse at http://localhost:8000
```

Do not open `index.html` directly as a `file://` URL — browsers block cross-origin
`fetch()` on file URIs, so PapaParse cannot load the CSV.

---

## Publishing Workflow

The site repo is at `~/git/vistadocs/vistadocs.github.io`, tracking
`git@github.com:vistadocs/vistadocs.github.io.git`.

```bash
# Copy updated files from working directory
cp ~/data/vista-docs/www/index.html ~/git/vistadocs/vistadocs.github.io/
cp ~/data/vista-docs/inventory/vdl_inventory_enriched.csv \
   ~/git/vistadocs/vistadocs.github.io/

# Commit and push
cd ~/git/vistadocs/vistadocs.github.io
git add index.html vdl_inventory_enriched.csv
git commit -m "Update VDL inventory table"
git push
```

GitHub Pages deploys automatically within ~60 seconds of the push.

**To update data only** (no HTML changes):

```bash
cp ~/data/vista-docs/inventory/vdl_inventory_enriched.csv \
   ~/git/vistadocs/vistadocs.github.io/
cd ~/git/vistadocs/vistadocs.github.io
git add vdl_inventory_enriched.csv
git commit -m "Refresh VDL inventory data"
git push
```

---

## File Locations

| Path | Description |
|---|---|
| `~/data/vista-docs/www/index.html` | Working copy of the web app |
| `~/data/vista-docs/inventory/vdl_inventory_enriched.csv` | Source data from pipeline |
| `~/git/vistadocs/vistadocs.github.io/` | GitHub Pages repo |
| `https://vistadocs.github.io` | Live site |
