Source of Paul's Blog
=====================

new page:

```
$ hugo new post/i-am-a-blogger.md
```

preview:

```
$ hugo server --theme=hugo-uno --buildDrafts --watch --buildFuture
```

save:

```
$ git add .
$ git commit -m 'new post'
$ git push origin master
```

generate site:

```
$ hugo --theme=hugo-uno --destination=../publish
$ cd ../publish
$ git add .
$ git commit -m "publish site"
$ git push origin master
```
