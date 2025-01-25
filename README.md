My blog
==================================

Responstry for my personal blog site which share information of rust, go and some reading notes.

This blog site build by [Hugo](https://gohugo.io) and use theme with [PaperMod](https://adityatelange.github.io/hugo-PaperMod/).

## Content

Using markdown syntax and support HTML, see Hugo [Content Formats](https://gohugo.io/content-management/formats/) for detail.

- [Common Markdown](https://commonmark.org)
- [Mermaid](https://mermaid-js.github.io/)
- [KaTeX](https://katex.org)

## Writing

Using hugo command crating a new post,

```bash
hugo new --kind post content/posts/<name>
```

or create a new markdown file directly.

### Requirements

- [Hugo](https://gohugo.io)

On macOS and install by brew command:

```bash
brew install hugo
```

### Run

```bash
hugo server
```

It will listen `:1313` port by default and watching local file change.

Visit blog site by URL: [http://localhost:1313/](http://localhost:1313/)

### Command

```bash
# List all posts
hugo list all

# List all drafts
hugo list drafts

# List all expired posts
hugo list expired

# List all posts dated in the future
hugo list future
```

See other command from hugo docs.
