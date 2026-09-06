# deno-gfm

Server-side GitHub-style Markdown rendering for Deno, including GitHub-style
CSS, syntax highlighting, and HTML sanitization.

> [!NOTE]\
> Community-maintained fork. This package is maintained independently by Simon
> Neutert and is based on denoland/deno-gfm. See the MIT license
> [LICENSE](./LICENSE) and changelog [CHANGELOG.md](./CHANGELOG.md) for
> attribution and changes.

## Current capabilities and integration paths

`deno-gfm` provides a batteries-included renderer for GitHub-style Markdown. The
current `render()` entry point works in Deno servers, command-line programs, and
build steps. Its import graph includes a server-oriented HTML sanitizer, so
bundling it directly for a browser may fail depending on the bundler. For a
client-side application, render the Markdown on the server or during a build,
then send the generated HTML and exported `CSS` to the browser.

`deno-gfm` does not currently parse MDX or hydrate interactive components. An
application that needs those features can compose the rendered HTML with a
framework or wrapper and let that integration manage components, islands, or
hydration.

A possible mix of `.md` and `.mdx` files is shown in
[github.com/simonneutert/deno-quickblog](https://github.com/simonneutert/deno-quickblog).

## Usage

First install the package with the command:

```sh
deno add jsr:@simonneutert/gfm
```

```js
import { CSS, render } from "@simonneutert/gfm";

const markdown = `
# Hello, world!

| Type | Value |
| ---- | ----- |
| x    | 42    |

\`\`\`js
console.log("Hello, world!");
\`\`\`
`;

const body = render(markdown, {
  baseUrl: "https://example.com",
});

const html = `
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
      main {
        max-width: 800px;
        margin: 0 auto;
      }
      ${CSS}
    </style>
  </head>
  <body>
    <main data-color-mode="light" data-light-theme="light" data-dark-theme="dark" class="markdown-body">
      ${body}
    </main>
  </body>
</html>
`;
```

### Styling

The GitHub CSS styles (https://primer.style) are used. There are two themes
available: light and dark.

There are three data attributes that can be used to control the theme:

- `data-color-mode`: `light` or `dark` or `auto`.
- `data-light-theme`: the name of the light theme (`light` or `dark`).
- `data-dark-theme`: the name of the dark theme (`light` or `dark`).

For example, if you want to use the dark theme only, set the following:

```html
<div data-color-mode="dark" data-dark-theme="dark" class="markdown-body">
  ... markdown body here ...
</div>
```

If you want to use the light or dark theme depending on the user's browser
preference, set the following:

```html
<div
  data-color-mode="auto"
  data-light-theme="light"
  data-dark-theme="dark"
  class="markdown-body"
>
  ... markdown body here ...
</div>
```

Also see the example application in the `example/` directory.

### Heading anchor icons

The bundled CSS hides the chain icon beside a heading until the heading is
hovered. To hide heading anchors entirely, load this override after the bundled
`CSS`:

```css
.markdown-body .anchor {
  display: none;
}
```

For different heading markup, extend `Renderer` and override its `heading()`
method. The CSS override is the direct option when only the icon's visibility
needs to change; a custom renderer is useful when the heading markup itself
needs to change.

## Public API examples

The examples below show how the public exports fit together.

### Rendering and customization

Pass `RenderOptions` to `render()` and extend `Renderer` when the generated
markup needs to change:

```ts
import {
  Marked,
  render,
  Renderer,
  type RenderOptions,
} from "@simonneutert/gfm";

class PlainHeadingRenderer extends Renderer {
  override heading({
    tokens,
    depth,
  }: Marked.Tokens.Heading): string {
    const text = this.parser.parseInline(tokens);
    return `<h${depth}>${text}</h${depth}>\n`;
  }
}

const options: RenderOptions = {
  breaks: true,
  renderer: new PlainHeadingRenderer(),
};

const html = render("# Hello\n\nFirst line\nsecond line", options);
```

### Plain-text and tokens

Use the stripping helpers for plain-text output or `Marked.lexer()` when the
complete Markdown token tree is needed:

```ts
import {
  type MarkdownSections,
  Marked,
  strip,
  stripSplitBySections,
} from "@simonneutert/gfm";

const markdown = "# Hello\n\nBody";
const text = strip(markdown);
const sections: MarkdownSections[] = stripSplitBySections(markdown);
const tokens = Marked.lexer(markdown);
```

### Stylesheets

`CSS` contains the bundled GitHub styles, while `KATEX_CSS` contains the
optional math styles. Their use is shown in [Usage](#usage) and
[Math rendering](#math-rendering).

## Extensibility

By default syntax highlighting for JavaScript, Markdown, and HTML is included.
You can include more languages importing them:

```js
import { CSS, render } from "@simonneutert/gfm";

// Add support for TypeScript, Bash, and Rust.
import "npm:prismjs@1.30.0/components/prism-typescript.js";
import "npm:prismjs@1.30.0/components/prism-bash.js";
import "npm:prismjs@1.30.0/components/prism-rust.js";
```

A full list of supported languages is available here:
https://unpkg.com/browse/prismjs@1.30.0/components/

Some Prism languages depend on other language components. Load those components
first. For example, PHP and PHPDoc require the following order:

````js
import { render } from "@simonneutert/gfm";

import "npm:prismjs@1.30.0/components/prism-markup-templating.js";
import "npm:prismjs@1.30.0/components/prism-php.js";
import "npm:prismjs@1.30.0/components/prism-javadoclike.js";
import "npm:prismjs@1.30.0/components/prism-phpdoc.js";

const html = render("```php\n<?php echo 'Hello';\n```");
````

Check Prism's `components.json` dependency metadata before importing another
language directly. Missing prerequisites can otherwise cause errors such as
`Cannot read properties of undefined (reading 'tokenizePlaceholders')`.

## Inline rendering

By default, all rendering is in blocks. There are cases where one would like to
render some inline markdown, and this is achievable using the `inline` setting:

```ts
import { render } from "@simonneutert/gfm";

const markdown = "My [Deno](https://deno.land) Blog";
const header = render(markdown, { inline: true });
console.log(header);
```

## Ordered lists with code blocks

Indent a fenced code block so it belongs to the preceding list item:

````md
1. Edit the file:

   ```ts
   const item = 1;
   ```

2. Continue with the next step.
````

Without the indentation, the fenced block ends the list and a later numbered
item starts a new list. When Markdown intentionally creates separate ordered
lists, `render()` preserves each list's explicit starting value.

## Parsing and plain-text output

Use `strip()` when an application needs one plain-text string instead of
rendered HTML. The returned string ends with a newline:

```ts
import { Marked, strip, stripSplitBySections } from "@simonneutert/gfm";

const markdown = "# Hello\n\nBody";
const text = strip(markdown); // "Hello\n\nBody\n"
const textSections = stripSplitBySections(markdown);
const tokens = Marked.lexer(markdown);
```

Image descriptions are retained in stripped output. Markdown images and raw HTML
`<img>` elements use their alt text. Image titles are supplementary and are not
included; an image without alt text does not contribute text.

`stripSplitBySections()` returns a smaller array of plain-text sections with
`header`, `depth`, and `content` fields. For applications that need the complete
Markdown token tree instead, the underlying Marked API is exported through
`Marked.lexer()`.

## Math rendering

By default math rendering is disabled. To enable it, you must include the
additional CSS and enable the `allowMath` setting:

```ts
import { CSS, KATEX_CSS, render } from "jsr:@simonneutert/gfm";

const markdown = `
Block math:

$$ y = x^2 $$

Inline math: $y = x^2$
`;

const body = render(markdown, {
  allowMath: true,
});

const html = `
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
      main {
        max-width: 800px;
        margin: 0 auto;
      }
      ${CSS}
      ${KATEX_CSS}
    </style>
  </head>
  <body>
    <main data-color-mode="light" data-light-theme="light" data-dark-theme="dark" class="markdown-body">
      ${body}
    </main>
  </body>
</html>
`;
```
