
# my Resume

# Run

```bash
docker run --name newblog --volume=".:/srv/jekyll" -p 3000:4000 -it jekyll/jekyll:3.5 jekyll serve --watch --drafts
```
```powershel
docker run  --volume="D:/Workspace/vinisdl.github.io/:/srv/jekyll" -p 3000:4000 -it jekyll/jekyll:3.5 jekyll serve --watch --drafts
```
* watch not work on windows