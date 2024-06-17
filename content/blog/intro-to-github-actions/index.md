---
date: "2019-08-29"
title: "Introduction to GitHub Actions"
categories: ["github","actions","devops","ci","cd"]
---

After what seems like an [eternity](https://github.blog/2018-10-17-action-demos/) I finally got added to the [GitHub Actions](https://github.com/features/actions/) beta a few days ago.

I recently created a [Hugo](https://gohugo.io) website for learning Kubernetes called [k8s.camp](https://k8s.camp) which is hosted in GitHub Pages and I figured that switching it from [CircleCI](https://circleci.com) to GitHub Actions would be a great way to learn Actions.

Before I get started I do want to mention that my experience as a new user trying to learn how actions work was sub-optimal.

My previous automation for [k8s.camp](https://k8s.camp) was using CircleCI which was very intuitive, I can't say the same for GitHub Actions.

## No good plan survives first contact

Obviously the first thing I did was google "github actions documentation" and the first hit was [developer.github.com/actions](https://developer.github.com/actions/) which I hastily clicked on to be greeted with the following:

![doc move deprecate](./doc-move.png)

Which suggested I was off to a rocky start. I clicked through to the ["Automating your workflow with GitHub Actions"](https://help.github.com/en/categories/automating-your-workflow-with-github-actions) page.

> Pretty much any time I googled a GitHub Actions question I was sent to the deprecated documentation with the `HCL` syntax rather than `yaml`.

Most of the existing examples are still in `HCL`, both in the documentation and out in the wild.

On top of this the examples in the documentation are quite trite and don't really help do anything useful. It's not even really clear how to structure the workflows and actions in the repo, so I had to fumble my way through it.

After a bunch of reading documentation and finding incomplete examples and blog posts I managed to slowly grok my way through it.

Hopefully this blog post will help others have a better first experience.

## Workflows and Actions

My understanding through trial and error is there are two things you need for GitHub Actions, Workflows and Actions.

### Workflows

Workflows are pretty much what you'd expect them to be. Files that describe the steps to be taken when an event is triggered.

These are stored in `.github/workflows` in your git repository and look something like this (in fact below is the [current workflow](https://github.com/paulczar/k8s-camp/blob/master/.github/workflows/build-and-deploy.yml) for k8s.camp):

```yaml
name: Build and Deploy to GitHub Pages
on:
  push:
    branches: [master]
jobs:
  build:
    name: build-and-deploy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1
      - uses: ./.github/actions/spell-check
      - uses: ./.github/actions/publish-gh-pages
        env:
          PUSH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

> Note: GitHub Actions act like GitHub Apps which means it gets it's own `GITHUB_TOKEN` secret which can be pulled into *environment variables* for the Actions. In fact there's a ton of default *environment variables* passed into the action such as the repository name and git commit sha.

You can see you define the type of activities to trigger the workflow (in this case a push to the `master` branch) and a set of steps to run. These steps call out to actions.

You can have multiple workflows based on different GitHub activities and even chain them together to create even more complex workflows.

## Actions

Actions are also pretty self explanatory, they're the discrete units of work required to fulfill each step in the workflow.

Actions can be local to the repo, or they can be called from another repository.

For example the above workflow calls a remote checkout action `uses: actions/checkout@v1` which literally looks out to the `v1` release of the github repo `actions/checkout` [here](https://github.com/actions/checkout/releases/tag/v1.0.0).

Actions are run inside Docker containers, and thus an Action is usually a `Dockerfile` combined with a script to run inside the container. Remote actions should contain an `action.yaml` to define the action and its interactions.

{{< twitter user="pczarkowski" id="1167437284410626054" >}}

You can vendor your actions into your repo (which I prefer to do, especially compared to pulling in some other random person's action blindly) in which case I prefer to place them in the `.github/actions` directory to keep things clean.

## The three steps

There are three steps defined in my workflow file above. These will be run in order by the action, if one fails the entire action will halt.

### Step 1 - checkout

The first step is using a github provided action to check out the source at the correct commit.

### Step 2 - spellcheck

The second step is running a spellcheck across all of my markdown files.

> As I mentioned an action runs in a Docker container, so an Action is generally defined by a `Dockerfile` to build the image, and an `ENTRYPOINT` script to run the action inside the image.

in the k8s.camp repo is the following local action `./.github/actions/spell-check` which consists of:

#### Dockerfile

This is a fairly basic Dockerfile that simply starts from a small `nodejs` image and adds in my entrypoint script.

> Note: the `LABEL`s are important to github actions and are documented [here](https://developer.github.com/actions/creating-github-actions/creating-a-docker-container/#label)

```dockerfile
FROM node:lts-alpine

LABEL "name"="Markdown Spell Checker"
LABEL "maintainer"="Paul Czarkowski"
LABEL "version"="0.0.1"

LABEL "com.github.actions.name"="Markdown Spell Checker"
LABEL "com.github.actions.description"="Markdown Spell Checker"
LABEL "com.github.actions.icon"="package"
LABEL "com.github.actions.color"="green"

COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

#### entrypoint.sh

The script is really simple and uses `npm` to install `markdown-spellcheck` locally and then runs it.

The exit code of `markdown-spellcheck` will be used to determine if the step passed or failed.

```bash
#!/bin/sh

cd $GITHUB_WORKSPACE

npm install markdown-spellcheck

./node_modules/markdown-spellcheck/bin/mdspell -r -n -a --en-us 'content/**/*.md'
```

### Step 3 - Publish to gh-pages

The third step will run `hugo` to generate the `html` for the website and then pushing the results to the `gh-pages` branch.

#### Dockerfile

Here we have a slightly more complicated `Dockerfile` that starts with Ubuntu and installs `curl`, `git`, and `hugo`

```dockerfile
FROM ubuntu:bionic

LABEL "name"="Publish k8s.camp"
LABEL "maintainer"="Paul Czarkowski"
LABEL "version"="0.0.1"

LABEL "com.github.actions.name"="Publish k8s.camp"
LABEL "com.github.actions.description"="Publish k8s.camp via gh-pages"
LABEL "com.github.actions.icon"="package"
LABEL "com.github.actions.color"="green"

RUN apt-get update > /dev/null && apt-get -yqq install curl git > /dev/null

RUN curl -sSL \
    https://github.com/gohugoio/hugo/releases/download/v0.56.3/hugo_0.56.3_Linux-64bit.tar.gz | \
    tar xzf - hugo && \
    chmod +x /hugo && \
    /hugo version

COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

#### entrypoint.sh

The entrypoint script configures git, checks out the `gh-pages` branch, and generates the html into that branch before pushing it back up to git using the `GITHUB_TOKEN` which is passed to it as the environment variable `PUSH_TOKEN`.

```bash
#!/bin/bash

set -o pipefail

echo "--> Configure git client"

git config --global user.email "username.taken@gmail.com"
git config --global user.name "Hugo Publisher"

echo "--> check out gh-pages"
git worktree add -B gh-pages public origin/gh-pages

echo "--> hugo publish"
cd $GITHUB_WORKSPACE
/hugo

echo "--> push gh-pages"
if [[ -z "$PUSH_TOKEN" ]]; then
  echo "No push token provided, skipping publish"
else
  cd public
  git add --all && \
  git commit -m "Github Action Build ${GITHUB_SHA} `date +'%Y-%m-%d %H:%M:%S'`" --allow-empty && \
  git remote set-url origin https://${GITHUB_ACTOR}:${PUSH_TOKEN}@github.com/${GITHUB_REPOSITORY}
  git push origin gh-pages:gh-pages
fi
```

> Note: I am using `git remote set-url` to update the git repo URL to include the `PUSH_TOKEN` which gives the action `push` access to the repository.

## End Result

Pushing this new `.github` directory structure defining a workflow and a set of steps up to my git repo is all I needed to set up the Actions, and the workflow triggered from this very first push.

Like all `ci` tools I had a dozen or so commits to stabilize the actual test scripts and workflow, but now all I have to do to publish a new version of the [k8s.camp](https://k8s.camp) website is to merge a commit into the `master` branch of the repo.

![screenshot of github actions](./actions2.png)

## Conclusion

Once you get through the initial hump of learning how GitHub actions work it becomes fairly straightforward to configure and use actions.

For the sake of sanity I have kept the workflow for [k8s.camp](https://k8s.camp) very simple, but I can see how you could quickly build out a fairly complex set of workflows to perform your CI/CD tasks.

While I kept the Actions in the local repository, I can see a very interesting versatility in having remote actions. I can see myself creating a central Actions github repo with all of my actions which I can then link back to from my various projects.

If you have a basic hugo setup publishing to `gh-pages` feel free to clone down the [paulczar/k8s-camp](https://github.com/paulczar/k8s-camp) repo and re-use them in your own.
