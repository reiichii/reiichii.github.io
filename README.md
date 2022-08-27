# Personal website

powered by [Hugo](https://gohugo.io) and [the Stack theme](https://github.com/CaiJimmy/hugo-theme-stack)

## Editing in a local machine

### Create posts file

```
hugo new post/2022-05-27-16/index.md
```

### Run the website

```
hugo server -D --gc
```

-D, --buildDrafts: include content marked as draft
--gc: enable to run some cleanup tasks (remove unused cache files) after the build