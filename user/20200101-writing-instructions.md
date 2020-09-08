---
title: "Writing instructions"
categories: "blogging"
abstract: ""
identity: "-1,-1"
---
## Metainfo

Use double-quotes (`"`) around the field values in the YAML header. You can escape double quotes with `\"` if you need them in the text.

```
---
title: "Writing \"instructions\"."
categories: "blogging,notes"
abstract: "Notes about formatting blog articles."
identity: "-1,-1"
---
```
 
## Do

 * ... use **empty lines around** headings, paragraphs, lists/enumerations, list items if they are complex, code blocks, quotes, and other block elements.
 * ... consider using an **extra empty line** before a new block, especially if the previous block has nested items.
 * ... **indent list items** with a single space, following four (4) spaces per each indentation level.
 * ... use `https` in hrefs if possible

## Don't

 * ... use `<h1>` (or `# Some title`) for a heading, go from `h2`'s in your blog posts.


## Editorial notes

 * Mark **broken links/pictures** with a footnote, or a note block if they need further explanation and/or alternate links.

 * **Link to other blog posts** using a relative link (to the blogs repo's root folder, starting with a `/`) to the `.md` file. Example:

    `/user/granicz/20061124-a-logo-interpreter-under-400-lines.md`

    Such links are then browseable in the repository and are also converted to `.html` links in the website.

 * ...


## Formatting

This is a test paragraph.

Styles: ~~strike through~~, ~subscript~ text, ^superscript^ text, ++inserted++ text, ==marked== text


## Tables

Right | Left | Default | Center
-----:|:-----|---------|:-----:
12    | 12   | 12      | 12
123   | 123  | 123     | 123
1     | 1    | 1       | 1


## Lists

Here is a bullet list:

 * Plain text
 * `Fixed font` text
 * ```fsharp
   let foo x = x+1
   ```
 * **Bold text**
 * *Italics text*

Here is the same list as a numbered list:

 1. Plain text
 2. `Fixed font` text
 3. ```fsharp
    let foo x = x+1
    ```
 4. **Bold text**
 5. *Italics text*

And now some text to test out different levels of heading:

## Code

```fsharp
let foo x = x+1
let bar x = foo x + 1
```


# Header 1

This is the biggest header, since it really stands out, don't use it in blog posts. We use it in other generated text where the largest header makes sense.

## Header 2 {#header2}

### Header 3

#### Header 4

##### Header 5

## Links

This is a link to [Header 2](#header2).


## Other stuff

> Here is a blockquote. It should be rendered as a block of text that stands out.

:::quote
This is a quote.
:::

:::note
This is a note.
:::

This is an example[^3] of a footnote.


[^3]: An example footnote.
