
# my Resume

# Run

```bash
export JEKYLL_VERSION=3.5
```

```bash
docker run --name newblog --volume="$PWD/src:/srv/jekyll" -p 3000:4000 -it jekyll/jekyll:$JEKYLL_VERSION jekyll serve --watch --drafts
```