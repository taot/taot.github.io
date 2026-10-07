# taot.github.io

Source of my research blog, built with [Hugo](https://gohugo.io) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.
GitHub Actions builds the site and deploys it to GitHub Pages on each push to `master`.

## Setup after clone

```bash
git submodule update --init --recursive
```

## Write a post

```bash
hugo new content posts/<slug>/index.md   # creates a draft page bundle
hugo server -D                           # preview with drafts at http://localhost:1313
```

Put images for the post in the same folder as `index.md`. Set `draft: false` to publish.

- Math: `$...$` inline, `$$...$$` display. KaTeX renders it at build time.
- Interactive HTML: put the file in `static/viz/`, then use `{{< viz src="viz/<file>.html" height="700px" >}}`.

## Check before push

```bash
hugo --gc --minify --environment production
```

## Maintenance

- Update Hugo locally, then set the same `HUGO_VERSION` in `.github/workflows/hugo.yml`.
- Update the theme: `git submodule update --remote --merge`.
