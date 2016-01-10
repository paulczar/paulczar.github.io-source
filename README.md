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

generate site:

```
$ hugo --theme=hugo-uno --destination=../publish


push to production:

```
$ git push origin master
$ hugo --theme=hugo-uno
$ ./deploy.sh
```
