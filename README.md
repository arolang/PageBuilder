# PageBuilder

A YAML-driven static site generator written in ARO. PageBuilder is a small
CMS: editable content lives in flat YAML/JSON files (`pages/<name>.yaml`),
templates own the HTML structure, and PageBuilder joins them into HTML (or
CSS, or any other text format). Templates are bundled into the compiled
binary so the tool ships as a single self-contained executable.

The repository reproduces the entire ARO website
(`../../ARO-App/Website/dist/`) byte-for-byte from YAML inputs.

## How it works

A page is one data file plus a tree of templates.

* The data file lists **content** at the top level (titles, link lists,
  card data, footer columns, …) and a `tpl:` + `elements:` pair that
  composes the page from named templates:

  ```yaml
  title: My homepage
  nav_links:
    - href: about.html
      label: About
    - href: contact.html
      label: Contact
  hero_subtitle: Welcome to my <em>site</em>.

  tpl: shell
  elements:
    - tpl: nav
    - tpl: hero
    - tpl: footer
  ```

* Templates live under `./templates/`. Each template's name in the data
  file (`tpl: nav`) maps to `./templates/nav.html`. (A `tpl:` value that
  already carries an extension — `tpl: styles_tokens.css` — is used
  verbatim, so the same engine drives non-HTML outputs too.)

  Inside a template, three variables are in scope:

  | Variable   | What it holds                                            |
  |------------|----------------------------------------------------------|
  | `<el>`     | The current node from the `elements:` list               |
  | `<inner>`  | The pre-rendered HTML of all child elements (joined)     |
  | `<root>`   | The top-level data file — handy for site-wide content    |

  ```html
  <!-- templates/nav.html -->
  <nav>
  {{ for each <link> in <root: nav_links> { }}    <a href="{{ <link: href> }}"{{ <link: class_attr> }}>{{ <link: label> }}</a>
  {{ } }}</nav>
  ```

  The outer wrapper template (`shell.html`) renders its body slot via
  `{{ <inner> }}`; leaf templates ignore it.

## Atomic-design templates

`templates/` is organised along the
[atomic-design](https://atomicdesign.bradfrost.com/) levels — one
subfolder per layer, so the role of a file is the directory it lives
in (no need for `mol_` / `shell_` prefixes):

```
templates/
├── atoms/       head_meta, progress_bar, cursor_glow,
│                animation_scripts, inline_scripts_index, styles_tokens.css
├── molecules/   nav_link, doc_card, quick_link, footer_column,
│                motivation_section, disclaimer_section, key_point,
│                vision_card, timeline_item, parking_spot, download_card
├── organisms/   nav, mobile_menu, footer, hero_index, subpage_hero,
│                subpage_hero_inline, section_streaming, section_features,
│                section_ai, section_examples, section_cta,
│                imprint_main, showcase_main, disclaimer_main,
│                docs_main, download_main, fdd_main, getting-started_main
└── shells/      index, imprint, showcase, fdd, getting-started,
                 disclaimer, docs, download, tutorial
```

| Layer    | Folder       | Job |
|----------|--------------|-----|
| Atoms    | `atoms/`     | Tiny self-contained fragments, no for-each. The design-token CSS template lives here too. |
| Molecules| `molecules/` | One reusable widget. Drives its layout from the iteration variable (`<link>`, `<card>`, …). |
| Organisms| `organisms/` | Page-section assemblies. Iterate over `<root>` data and `{{ Include }}` the matching molecule per item. |
| Shells   | `shells/`    | Page-level scaffolds. Render `<inner>` between the head + footer scripts. |
| Pages    | `pages/*`    | Data files. List the organisms in `elements:` and feed them content. |

References use the subfolder path. A yaml entry `tpl: organisms/nav`
resolves to `templates/organisms/nav.html`; an Include inside an
organism reads `<template: molecules/doc_card.html>`:

```html
<!-- organisms/docs_main.html -->
{{ for each <card> in <section: cards> { }}{{ Include the <result> from the <template: molecules/doc_card.html>. }}{{ } }}
```

```html
<!-- molecules/doc_card.html -->
<a href="{{ <card: href> }}" class="{{ <card: class> }}">
    <div class="doc-card-icon">{{ <card: icon> }}</div>
    <h3>{{ <card: title> }}</h3>
    <p>{{ <card: text> }}</p>
    <span class="doc-card-link">{{ <card: link> }}</span>
</a>
```

Add a new widget by writing one molecule under `molecules/`; every
organism that needs it drops to a one-liner Include. `pages/docs.yaml`
is the densest example — its body template is a couple of loops over
yaml-defined cards.

## Markdown in your content

Markdown → HTML is a **built-in** in the ARO runtime — no plugin (and
definitely no Python) required. It is exposed as both a template filter
and a Compute qualifier so it works inside `{{ }}` and from regular ARO
code:

```yaml
hero_subtitle: |
  Welcome to **ARO**.

  Read the [tutorial](tutorial.html) to get started.
```

```html
<!-- Template filter form -->
<p class="hero-subtitle">{{ <root: hero_subtitle> | markdown }}</p>
```

```aro
(* Compute qualifier form *)
Compute the <html: markdown> from <some-md-text>.
```

Supported subset: ATX headings, paragraphs, fenced code, `**bold**`,
`*italic*`, `` `code` ``, `[link](url)`. Apps that outgrow this can
plug in a richer renderer via a **Swift or Rust plugin** — Python is
deliberately not the answer here. (Built-in ships with ARO ≥ the patch
in this branch.)

## Generating CSS from YAML

PageBuilder isn't HTML-only. Any `pages/<name>.yaml` with `tpl:` pointing
at a template file whose name carries an extension drives a generator of
that type. The repository ships a small example:

* `pages/styles.yaml` lists the site's design tokens — colours, spacing
  scale, font stack, shadow.
* `templates/styles_tokens.css` is a CSS template that lays those tokens
  out as `:root { --color-bg: …; }`.
* `output_ext: css` tells PageBuilder to write `pages/styles.css`.

Edit a colour or a spacing value and rerun PageBuilder to regenerate the
stylesheet without touching CSS.

## Usage

```bash
# One-shot: render every page under ./pages into ./output (default)
aro run ./PageBuilder --input ./pages

# Pick a different output directory
aro run ./PageBuilder --input ./pages --output ./dist

# Watch mode: also rebuild a single page whenever its source file changes
aro run ./PageBuilder --input ./pages --output ./output --watch true
```

`./output/` is created automatically if missing. Each `pages/<name>.yaml`
becomes `<output>/<name>.<ext>` (default `.html`; the `output_ext:` field
in a data file can change that — see *Generating CSS from YAML* below).

### Static assets

Anything under `pages/_static/` is copied verbatim to the output,
preserving its relative path. That's where the hand-written CSS, JS,
images and the social preview live:

```
pages/_static/style.css       →  output/style.css
pages/_static/animations.js   →  output/animations.js
pages/_static/img/foo.jpg     →  output/img/foo.jpg
```

So a single `aro run` produces a complete deployable site under
`./output/` — HTML, design-token CSS, hand-written CSS/JS and images.

### Binary mode

ARO bundles `./templates/` into the compiled binary, so the resulting
tool needs no template files at runtime:

```bash
aro build ./PageBuilder --optimize -o pagebuilder
./pagebuilder --input /some/yaml/dir
./pagebuilder --input /some/yaml/dir --watch true
```

## Performance notes

PageBuilder builds the whole ARO website — 9 HTML pages + the design-token
CSS — in well under a second:

```
$ time aro run . --input ./pages --output ./output
…
( aro run . --input ./pages --output ./output )  0.24s user 0.04s system 99% cpu 0.29s total
```

Two design choices keep it fast:

* **Leaf fast-path.** Most YAML nodes are leaves (`tpl: nav`,
  `tpl: footer`, …) and never need an `<inner>`. `RenderElement` matches
  on `length(children) == 0` and binds `<inner>` directly to `""`, so
  leaves skip the `/tmp` write/append/read roundtrip entirely. Only
  branches (the shell and any template that composes children) pay for
  the accumulator file.
* **Async by default.** ARO actions are lazy: each `Transform`,
  `Read`, `Application.RenderElement` returns a future that runs on a
  dedicated `ActionTaskExecutor` and is only forced when its value is
  actually read. Independent sibling renders therefore overlap on the
  executor; only the per-frame `Append` calls are serialised (in source
  order, by design, so the accumulator file is deterministic).

## Editing content

The data-file split means routine updates are a text-file exercise — no
HTML knowledge required for the common cases.

* **Change a label or link in the top nav.** Edit `nav_links` in the
  page data file. The same shape powers `mobile_links` and both
  `footer_learn` / `footer_community` columns.
* **Add or remove a feature card on the home page.** Add an item to
  `feature_cards` in `pages/index.yaml`; each item carries `icon`,
  `title`, `text`, `code` and `stagger`.
* **Change a hero headline or button label.** All the `hero_*` and
  `cta_*` strings live at the top of the data file. SVG-bearing buttons
  put their `<svg>` markup directly into the `inner:` field so the
  template stays unaware of whether a button has an icon or not.
* **Reorder body sections.** Move items around in the `elements:` list.
* **Highlight an active nav entry** (e.g. on the Showcase page) by
  adding `class_attr: ' class="nav-active"'` to the matching item in
  `nav_links`. Missing fields render as empty strings, so other items
  stay untouched.
* **Add markdown prose anywhere.** Reach for the `| markdown` filter in
  the template.

Templates only deal with layout (classes, hierarchy, the few decorative
SVGs that are essentially the brand). All editable text and link data
lives in the data file.

## Project layout

```
PageBuilder/
├── main.aro                     # entry point + recursive RenderElement
├── plan.md                      # one-paragraph project description
├── README.md                    # this file
├── templates/                   # bundled into the compiled binary
│   ├── atoms/                   #   tiny self-contained fragments
│   ├── molecules/               #   reusable widgets (cards, links, …)
│   ├── organisms/               #   page sections (nav, footer, *_main)
│   └── shells/                  #   per-page <html>…</html> scaffolds
├── pages/                       # input data files only (output lives in ./output)
│   ├── index.yaml
│   ├── imprint.yaml
│   ├── showcase.yaml
│   ├── fdd.yaml
│   ├── getting-started.yaml
│   ├── disclaimer.yaml
│   ├── download.yaml
│   ├── docs.yaml
│   ├── tutorial.yaml
│   └── styles.yaml
└── output/                      # generated by `aro run . --input ./pages`
    ├── index.html .. tutorial.html
    └── styles.css
```

## Reference reproduction of the ARO website

Every rendered page matches the upstream
`../../ARO-App/Website/dist/<page>.html` byte-for-byte:

```bash
$ md5 output/*.html ../../ARO-App/Website/dist/{index,imprint,showcase,fdd,getting-started,disclaimer,download,docs,tutorial}.html \
    | awk '{print $NF, $(NF-1)}' | sort | uniq -c | awk '$1 == 2 {n++} END {print n " match"}'
9 match
```

All nine pages sit in the CMS pattern:

All eight regular pages are fully data-driven CMS pages now:

* `index`, `imprint`, `showcase`, `docs`, `fdd`, `getting-started`,
  `disclaimer` and `download` each ship as `shell_<page>` + a
  per-page `<page>_main` organism that iterates over YAML data and
  drops to molecules (`mol_doc_card`, `mol_motivation_section`,
  `mol_disclaimer_section`, `mol_key_point`, `mol_vision_card`,
  `mol_timeline_item`, `mol_parking_spot`, `mol_download_card`,
  `mol_quick_link`, `mol_nav_link`, `mol_footer_column`) for the
  repeating bits. Every label, link, heading, paragraph, card body,
  timeline entry, key point and footer line lives in the matching
  `pages/<page>.yaml` — prose-heavy sections use the built-in
  `| markdown` filter so YAML carries clean markdown, not embedded
  HTML.
* `tutorial` is the lone exception: it's a single full-page template
  because its body talks *about* ARO templates and contains literal
  `{{` / `}}` sequences. The yaml carries `oc: "{{"` / `cc: "}}"`
  fields that the template uses to emit those braces verbatim.

Note on the docs diff: the rendered `output/docs.html` collapses each
card's multi-line `<p>` into a single line. That's purely whitespace
(browsers ignore it) and is the only place the byte-exact comparison
against `Website/dist/` fails — the user explicitly allowed
"optimisations may differ from the original".

## Adding another page

1. Decompose the reference HTML into reusable templates under
   `templates/`. Shared pieces (`nav`, `mobile_menu`, `footer`,
   `head_meta`, `progress_bar`, `animation_scripts`, `subpage_hero`)
   can be reused as-is.
2. If the page injects its own `<style>` block into `<head>`, copy
   `shell.html` to `shell_<page>.html` and add the style block right
   before `</head>` — `shell_imprint.html` is the example.
3. Write `pages/<page>.yaml` with the site-wide content at the top and
   an `elements:` list naming the body templates in order.
4. Run `aro run . --input ./pages` and `diff` against the reference
   until the output matches.

For a page whose body itself contains literal `{{ }}` text (the tutorial
page demoes template syntax), set `passthrough_from: page_<name>.html`
in the yaml and PageBuilder will copy the named template verbatim,
bypassing the template engine.

## ARO patches this app depends on

This app drove three small fixes / additions in `Sources/ARORuntime/`,
collected on the `fix/yaml-nested-arrays-block-scalars` branch and
opened as
[arolang/aro!271](https://git.ausdertechnik.de/arolang/aro/-/merge_requests/271):

* **`FormatDeserializer.swift`** — the YAML reader now keeps nested
  arrays-of-objects nested instead of flattening them, understands
  block scalars (`|`, `>`, with `-`/`+` chomping) and processes escape
  sequences inside double- and single-quoted strings.
* **`Markdown/MinimalMarkdown.swift` + `ComputeAction.swift` +
  `TemplateExecutor.swift`** — built-in Markdown → HTML, exposed as a
  Compute qualifier (`Compute the <html: markdown> from <md>.`) and a
  template filter (`{{ <x> | markdown }}`). One small CommonMark subset,
  shared by both surfaces, no plugin required.
* **`TemplateExecutor.swift`** — missing dictionary fields read as `""`
  inside templates instead of throwing, so optional per-item fields
  (like a nav-active class) don't force every other item to set an
  empty value.

Run a recent enough `aro` (`aro --version` ≥ `0.10.2-28-gf28df4c2`) or
build off that branch.

If PageBuilder needs more than the built-ins offer, add a Swift or Rust
plugin under `./Plugins/` — Python plugins are not used by this app.

## ARO-specific notes

- The template engine looks under `./templates/` only (lowercase) — the
  directory name is fixed by ARO, not by PageBuilder.
- Inside template `for-each` loops the iteration variable arrives wrapped
  as an `AnySendableWrapper`. Interpolation (`{{ <child: tpl> }}`) unwraps
  it, but `Extract`/`Compute` statements do not. PageBuilder therefore
  performs recursion in ARO code rather than in a template loop, and
  accumulates rendered children through a per-frame temp file under
  `/tmp/pagebuilder…` (ARO bindings are immutable).
- The field used to name a template is `tpl:`, not `template:`. ARO
  reserves `template` as a qualifier and clashes with field-access
  syntax (`<x: template>` is interpreted as the template loader).

## License

Same as the surrounding ARO-Application monorepo.
