# Offsec Journal

Source for https://iamzhanteiu.github.io, built with Hugo and the PaperMod theme.

## Write a post

```sh
git clone --recurse-submodules https://github.com/iamzhanteiu/iamzhanteiu.github.io.git
hugo new content posts/<slug>.md   # creates a draft from archetypes/posts.md
hugo server -D                     # preview at http://localhost:1313, drafts included
```

Set `draft: false` and push to `main`. The `Deploy Hugo site to Pages` workflow builds and publishes the site.

Publish only reports that are disclosed on HackerOne or cleared by the program.
