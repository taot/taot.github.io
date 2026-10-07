# taot.github.io

Source of my research blog, built with [Hugo](https://gohugo.io) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.
GitHub Actions builds the site and deploys it to GitHub Pages on each push to `master`.

## Setup after clone

```bash
git submodule update --init --recursive
```

## Write a post

```bash
hugo new content posts/YYYY/MM/<slug>/index.md   # creates a draft page bundle, e.g. posts/2026/10/my-topic/index.md
hugo server -D                                   # preview with drafts at http://localhost:1313
```

Posts go in year/month folders. The URL is the same as the folder, for example `/posts/2026/10/my-topic/`.
Do not add `_index.md` to the year or month folders.

Put images and other files for the post in the same folder as `index.md`. Set `draft: false` to publish.

- Math: `$...$` inline, `$$...$$` display. KaTeX renders it at build time.
- Interactive HTML: put the file in the post folder, then use `{{< viz src="<file>.html" height="700px" wide="true" >}}`.

## Check before push

```bash
hugo --gc --minify --environment production
```

## Maintenance

- Update Hugo locally, then set the same `HUGO_VERSION` in `.github/workflows/hugo.yml`.
- Update the theme: `git submodule update --remote --merge`.
