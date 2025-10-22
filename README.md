My blog
==================================

Responstry for my personal blog site which share information of rust, go and some reading notes.

This blog site build by [Zola](https://www.getzola.org) and use theme with [apollo](https://not-matthias.github.io/apollo/).

## Content

Writing post in content directory, and it's using markdown syntax and support HTML, mermaid and KaTex, and see more explain in [docs](https://www.getzola.org/documentation/content/overview/).

- [Common Markdown](https://commonmark.org)
- [Mermaid](https://mermaid-js.github.io/)
- [KaTeX](https://katex.org)

Create new post with command as follow:

```bash
touch ./content/posts/new-post.md
```

## Requirements

- [Zola](https://www.getzola.org)

On macOS and install by brew command:

```bash
brew install zola
```

### Run

```bash
zola serve
```

It will listen `:1111` port by default and watching local file change.

Visit blog site by URL: [http://localhost:1111/](http://localhost:1111/)

### Command

```bash
$ zola --help
A fast static site generator with everything built-in

Usage: zola [OPTIONS] <COMMAND>

Commands:
  init        Create a new Zola project
  build       Deletes the output directory if there is one and builds the site
  serve       Serve the site. Rebuild and reload on change automatically
  check       Try to build the project without rendering it. Checks links
  completion  Generate shell completion
  help        Print this message or the help of the given subcommand(s)

Options:
  -r, --root <ROOT>      Directory to use as root of project [default: .]
  -c, --config <CONFIG>  Path to a config file other than config.toml in the root of project [default: config.toml]
  -h, --help             Print help
  -V, --version          Print version
```

See CLI usage from [docs](https://www.getzola.org/documentation/getting-started/cli-usage/).
