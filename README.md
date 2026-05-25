# PageBuilder

A YAML-driven static site generator written in ARO. PageBuilder reads YAML
files that describe pages as a tree of templates and renders them to HTML.
Templates are bundled into the compiled binary so the whole tool ships as a
single self-contained executable.

## How it works

Each `.yaml` file is a tree of nodes. Every node names a template from
`./templates/` (bare name, no extension — `page_base` resolves to
`./templates/page_base.html`) and may carry arbitrary parameters plus an
`elements:` list of child nodes.

```yaml
tpl: page_base
title: My first page
elements:
  - tpl: header
    elements:
      - tpl: navigation
  - tpl: body
    elements:
      - tpl: hero
        headline: Welcome
        image: ./dog.jpg
      - tpl: content
        text: |
          Some **markdown** goes here.
  - tpl: footer
```

Inside a template the current node is bound to `<el>` and any field is
reachable as `{{ <el: fieldName> }}`. All child elements are pre-rendered
depth-first and joined into a single string exposed as `{{ <inner> }}`:

```html
<!-- templates/page_base.html -->
<!DOCTYPE html>
<html>
<head><title>{{ <el: title> }}</title></head>
<body>
{{ <inner> }}</body>
</html>
```

A leaf template simply ignores `<inner>`.

## Usage

```bash
# One-shot: build every *.yaml under ./pages into a sibling *.html
aro run ./PageBuilder --input ./pages

# Watch mode: also rebuild a single page whenever its yaml changes
aro run ./PageBuilder --input ./pages --watch true
```

The output mirrors the input directory structure — `pages/blog/post.yaml`
becomes `pages/blog/post.html`.

### Binary mode

ARO bundles `./templates/` into the compiled binary, so the resulting tool
needs no template files at runtime:

```bash
aro build ./PageBuilder --optimize -o pagebuilder
./pagebuilder --input ./some/yaml/dir
./pagebuilder --input ./some/yaml/dir --watch true
```

## Project layout

```
PageBuilder/
├── main.aro          # Application-Start, BuildAll, BuildOne handler,
│                     # RenderElement, watch-mode handler
├── plan.md           # One-paragraph project description
├── README.md         # This file
├── templates/        # All page fragments (bundled into the binary)
│   ├── shell.html        # outer scaffold for the index page
│   ├── shell_imprint.html# scaffold with imprint-specific <style>
│   ├── head_meta.html    # the shared <head> meta block
│   ├── nav.html          # main navigation
│   ├── mobile_menu.html  # mobile drawer
│   ├── hero_index.html   # index-page hero with the plugins SVG
│   ├── subpage_hero.html # generic <header class="subpage-hero">
│   ├── section_*.html    # individual content sections of the index page
│   ├── imprint_main.html # imprint body
│   ├── progress_bar.html # scroll progress indicator
│   ├── footer.html       # site footer
│   ├── animation_scripts.html  # <script src="animations.js">
│   ├── inline_scripts_index.html # index-page inline JS
│   ├── cursor_glow.html  # cursor glow effect div
│   └── ...               # add more as you build more pages
└── pages/            # input + output live side-by-side
    ├── index.yaml    # source
    ├── index.html    # generated (byte-exact match of the ARO website)
    ├── imprint.yaml
    └── imprint.html
```

## Reference reproduction of the ARO website

PageBuilder was bootstrapped against the existing ARO website
(`../../ARO-App/Website/dist/`) as the reference output. Each rendered
page is required to match the upstream byte-for-byte. Use `diff` to
verify:

```bash
diff pages/index.html ../../ARO-App/Website/dist/index.html
diff pages/imprint.html ../../ARO-App/Website/dist/imprint.html
```

`md5` checksums on a successful run agree exactly with the upstream files.

### Adding another page

1. Decompose the reference HTML into reusable templates under
   `templates/`. Templates that already exist (`nav`, `mobile_menu`,
   `footer`, `head_meta`, `progress_bar`, `animation_scripts`,
   `subpage_hero`) can be shared as-is.
2. If the page injects its own `<style>` block into `<head>`, copy
   `shell.html` to `shell_<page>.html` and add the style block right
   before `</head>`.
3. Write `pages/<page>.yaml` listing the templates in body order.
4. Run `aro run . --input ./pages` and `diff` against the reference until
   the output matches byte-for-byte.

## ARO-specific notes

- The template engine looks under `./templates/` only (lowercase) — the
  directory name is fixed by ARO, not by PageBuilder.
- Inside template `for-each` loops the iteration variable arrives wrapped
  as an `AnySendableWrapper`. Interpolation (`{{ <child: tpl> }}`) unwraps
  it, but `Extract`/`Compute` statements do not. PageBuilder therefore
  performs recursion in ARO code rather than in a template loop, and
  accumulates rendered children through a per-frame temp file under
  `/tmp/pagebuilder…` because ARO bindings are immutable.
- The yaml field for the template name is `tpl:`, not `template:`,
  because `template` is a reserved qualifier name in ARO and clashes with
  field-access syntax (`<x: template>`).

## License

Same as the surrounding ARO-Application monorepo.
