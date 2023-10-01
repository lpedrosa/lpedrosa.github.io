# My blog

Running this locally:

```sh
# bind on all so I can check how it looks on my phone
hugo server --bind 0.0.0.0

# to also show draft posts
hugo server -D --bind 0.0.0.0
```

Adding a new blog post:

```sh
hugo new content blog/<title-of-the-blogpost>.md
```
