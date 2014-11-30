Source of Paul's Blog
=====================

The source branch is where you do the work.

The public/ directory is a git subtree back to the master branch which is where the github user page is hosted from.

edit the source branch and then

push to production:

```
$ git subtree push --prefix=public git@github.com:paulczar/paulczar.github.io master
```
