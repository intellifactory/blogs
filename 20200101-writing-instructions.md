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

 * Mark **broken links/pictures** with a footnote, or a note block if they need further explanation and/or alternate links. Example:

    ````
    > **Review note** <br />
    > This resource needs to be re-added.
    > ![](https://intellifactory.com/ShowDigitalAsset.aspx?DigitalAsset=140)
    ````

 * **Link to other IntelliFactory blog posts** using a relative link (to the blogs repo's root folder, starting with a `/`) to the `.md` file. Example:

    `/user/granicz/20061124-a-logo-interpreter-under-400-lines.md`

    Such links are then browseable in the repository and are also converted to `.html` links in the website.

    This means converting the following types of links to repo links:
    
     * `https://forums.websharper.com/blog/*/*`
     * `http://websharper.com/blog-entry/*/*`
     * `http://fpish.net/blog/*/id/*/*`

 * If a link has gone dead but a newer version exists, update to that with a footnote:

    ```   
    [^1]: This link has been updated to point to WebSharper reactive forms, which include further pointers on formlets.
    ```   

 * If a link is dead and has no direct substitute, give your best alternative in the footnote. Example:

    ```   
    [^1]: This link is dead and has been removed. You can find formlet examples on [Try WebSharper](https://try.websharper.com), filtering for snippets that use `UI.Formlets`. These formlets are an enhanced, reactive version of the original formlet library. You can find more information the [WebSharper.Forms README](https://github.com/dotnet-websharper/forms).
    ```

 * If a link is dead and no alternatives exist, try locating an archived version of it, and mark it accordingly with a footnote. Example:

    ```
    [^1]: This link is no longer available. A (broken) snapshot of the page from the Internet archives is available [here](https://web.archive.org/web/20101123072954/http://intellifactory.com/products/wsp/samples/ExtJs.aspx).
    ```


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
