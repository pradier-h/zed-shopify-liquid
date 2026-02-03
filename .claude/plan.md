# Zed Shopify Liquid Extension Enhancement Plan

## Overview
Enhance the Zed Liquid extension to provide better parsing inside Liquid tags and enable go-to-definition (ctrl+click) for snippets, sections, and other references.

## Current State
- Basic syntax highlighting via `highlights.scm`
- HTML/JS/CSS/JSON injection via `injections.scm`
- LSP integration launching `shopify theme language-server`
- Basic bracket matching

## Key Finding
The Shopify language server **already supports** go-to-definition, find references, hover, and completion. The issue is that these features may not be properly enabled due to missing LSP configuration.

---

## Implementation Tasks

### Phase 1: Fix LSP Configuration (High Priority)

#### 1.1 Add `language_server_initialization_options` to `src/liquid.rs`
The extension is missing initialization options that enable LSP features.

```rust
fn language_server_initialization_options(
    &mut self,
    _language_server_id: &zed::LanguageServerId,
    worktree: &zed::Worktree,
) -> Result<Option<serde_json::Value>> {
    let settings = LspSettings::for_worktree("liquid", worktree)
        .ok()
        .and_then(|lsp_settings| lsp_settings.initialization_options.clone())
        .unwrap_or_default();
    
    Ok(Some(settings))
}
```

#### 1.2 Update workspace configuration structure
Current code wraps settings in `{"liquid": settings}`. The Shopify LSP may expect a different structure with `themeCheck` configuration:

```rust
fn language_server_workspace_configuration(
    &mut self,
    _language_server_id: &zed::LanguageServerId,
    worktree: &zed::Worktree,
) -> Result<Option<serde_json::Value>> {
    let settings = LspSettings::for_worktree("liquid", worktree)
        .ok()
        .and_then(|lsp_settings| lsp_settings.settings.clone())
        .unwrap_or_default();

    Ok(Some(serde_json::json!({
        "themeCheck": {
            "checkOnOpen": true,
            "checkOnChange": true,
            "checkOnSave": true
        },
        "liquid": settings
    })))
}
```

#### 1.3 Add language ID mapping in `extension.toml`
```toml
[language_servers.liquid.language_ids]
"Liquid" = "liquid"
```

---

### Phase 2: Improve Syntax Highlighting

#### 2.1 Enhance `highlights.scm` with missing patterns

Add these patterns for better in-tag highlighting:

```scm
; Variable definitions - highlight assigned variable names
(assignment_statement
  variable_name: (identifier) @variable.definition)

(capture_statement
  variable: (identifier) @variable.definition)

; For loop variable highlighting
(for_loop_statement
  item: (identifier) @variable.parameter)

(tablerow_statement
  item: (identifier) @variable.parameter)

; Property access chains (product.title, item.price)
(access
  receiver: (identifier) @variable
  property: (identifier) @property)

(access
  receiver: (access) @variable
  property: (identifier) @property)

; Render/section file references - special string highlighting
(render_statement
  file: (string) @string.special)

(section_statement
  (string) @string.special)

(sections_statement
  (string) @string.special)

(include_statement
  (string) @string.special)

; Named arguments in function calls
(argument
  key: (identifier) @variable.parameter)

; Range expression
(range
  "(" @punctuation.bracket
  ".." @operator
  ")" @punctuation.bracket)
```

---

### Phase 3: Add New Query Files

#### 3.1 Create `languages/liquid/outlines.scm` for document symbols
```scm
; Capture blocks show in outline
(capture_statement
  variable: (identifier) @name) @item

; Schema sections
(schema_statement) @item

; For loops (structural navigation)
(for_loop_statement
  item: (identifier) @name) @item
```

#### 3.2 Create `languages/liquid/folds.scm` for code folding
```scm
[
  (if_statement)
  (unless_statement)
  (case_statement)
  (for_loop_statement)
  (capture_statement)
  (form_statement)
  (paginate_statement)
  (tablerow_statement)
  (raw_statement)
  (javascript_statement)
  (schema_statement)
  (style_statement)
  (comment)
] @fold
```

---

### Phase 4: Update Language Configuration

#### 4.1 Enhance `languages/liquid/config.toml`
Add additional brackets and word characters:

```toml
name = "Liquid"
grammar = "liquid"
path_suffixes = ["liquid"]
brackets = [
    { start = "{{", end = "}}", close = true, newline = false },
    { start = "{{-", end = "-}}", close = true, newline = false },
    { start = "{%", end = "%}", close = true, newline = false },
    { start = "{%-", end = "-%}", close = true, newline = false },
    { start = "(", end = ")", close = true, newline = false },
    { start = "[", end = "]", close = true, newline = false },
    { start = "'", end = "'", close = true, newline = false, not_in = ["string", "comment"] },
    { start = "\"", end = "\"", close = true, newline = false, not_in = ["string", "comment"] },
]
autoclose_before = ";:.,=}])> \n\t"
block_comment = ["{% comment %}", "{% endcomment %}"]
word_characters = ["-", "_"]
scope_opt_in_language_servers = ["tailwindcss-language-server"]
```

---

## Files to Modify

| File | Changes |
|------|---------|
| `src/liquid.rs` | Add `language_server_initialization_options`, update workspace config |
| `extension.toml` | Add language_ids mapping |
| `languages/liquid/highlights.scm` | Add variable definitions, property access, argument highlighting |
| `languages/liquid/config.toml` | Add brackets, word_characters, scope_opt_in |
| `languages/liquid/outlines.scm` | **New file** - document outline queries |
| `languages/liquid/folds.scm` | **New file** - code folding queries |

---

## Verification

### Test Go-to-Definition
1. Create a Shopify theme project with:
   - `snippets/product-card.liquid`
   - `sections/featured-products.liquid` containing `{% render 'product-card' %}`
2. Open `featured-products.liquid` in Zed
3. Ctrl+click on `'product-card'` - should navigate to the snippet file

### Test Syntax Highlighting
Create a test file with:
```liquid
{% assign my_variable = product.title | upcase %}
{% for item in collection.products limit: 10 %}
  {% render 'product-card', product: item, show_price: true %}
  {{ item.price | money }}
{% endfor %}
```
Verify:
- `my_variable` highlighted as variable definition
- `item` in for loop highlighted as parameter
- `product.title` shows `product` as variable, `title` as property
- `'product-card'` highlighted as special string
- `product:`, `show_price:` highlighted as parameters

### Test LSP Features
- Hover over a Liquid filter (e.g., `upcase`) - should show documentation
- Type `{{ product.` - should show completion suggestions
- Check for diagnostics/linting warnings

---

## Risks & Mitigations

1. **LSP not responding to definition requests**: May need to debug LSP communication. Use Zed's developer tools to inspect LSP traffic.

2. **Theme structure required**: Go-to-definition only works in proper Shopify theme projects with `snippets/`, `sections/`, etc. directories.

3. **Tree-sitter query compatibility**: Test queries against actual files to ensure they match expected nodes.
