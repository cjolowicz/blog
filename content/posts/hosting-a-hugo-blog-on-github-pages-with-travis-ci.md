---
title: "Hosting a Hugo blog on GitHub Pages with Travis CI"
date: 2019-04-27T10:48:29+02:00
tags: ["hugo", "github-pages", "travis"]
---

This post describes how to set up a blog using [Hugo](https://gohugo.io/), an
open-source static site generator. The blog is hosted on [GitHub
Pages](https://pages.github.com/), a web hosting service offered by GitHub. The
[Travis CI](https://travis-ci.com) continuous integration service is used to
deploy changes to the blog.

> This post is based on Artem Sidorenko's article
> [Hugo on GitHub Pages with Travis CI](https://www.sidorenko.io/post/2018/12/hugo-on-github-pages-with-travis-ci/).

**Contents**

- [Overview](#overview)
- [Installing Hugo](#installing-hugo)
- [Setting up the blog repository](#setting-up-the-blog-repository)
- [Setting up the github.io repository](#setting-up-the-githubio-repository)
- [Continuous Deployment](#continuous-deployment)
- [Writing a post](#writing-a-post)
- [Links](#links)

## Overview

Running a Hugo blog on GitHub Pages requires you to set up two GitHub
repositories:

- The first repository is named `blog` and holds the Hugo sources.
- The second repository is named `username.github.io` and holds the generated
  content.

(Throughout this post, replace _username_ with your GitHub username.)

You also need to set up Travis CI such that, when you push a change to `blog`,
it invokes Hugo to rebuild the site, and pushes the generated content to
`username.github.io`. GitHub Pages will then deploy the site to
https://username.github.io/.

## Installing Hugo

Installing [Hugo](https://gohugo.io/) on macOS is easily achieved using
[Homebrew](https://brew.sh/):

```sh
brew install hugo
```

See [Install Hugo](https://gohugo.io/getting-started/installing/) for
alternatives.

## Setting up the blog repository

In this section you set up the `blog` repository on GitHub.

#### Creating the blog

The first step is to generate the files for the new Hugo site:

```sh
hugo new site blog
```

#### Creating the repository

Initialize a git repository in the newly created directory, and create the
initial commit:

```sh
cd blog

git init
git add .
git commit -m "Initial commit"
```

#### Installing a theme

The next step is to install a theme. For now, stick with the
[ananke](https://themes.gohugo.io/gohugo-theme-ananke/) theme recommended in
Hugo's [Quick Start](https://gohugo.io/getting-started/quick-start/) tutorial.

```sh
git submodule add \
    https://github.com/budparr/gohugo-theme-ananke.git \
    themes/ananke
git add .
git commit -m "Add submodule themes/ananke"
```

This command adds the _ananke_ git repository to the `themes` subfolder. Using a
[git submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules) has the
advantage of allowing you to track upstream changes to the theme.

#### Configuring the site

Hugo is configured using a file called `config.toml`, which has already been
generated for us. Edit this file to set the site URL and the blog title, and to
declare the theme you just installed. This is what the file should look like:

```toml
baseURL = "https://username.github.io/"
languageCode = "en-us"
title = "My Blog"
theme = "ananke"
```

Commit your changes:

```sh
git add config.toml
git commit -m "Configure site"
```

#### Publishing the repository

You are now ready to publish the `blog` repository to GitHub. One convenient way
to do so is using [hub](https://github.com/github/hub), a command-line tool for
managing GitHub repositories:

```sh
brew install hub
hub create
git push origin master
```

## Setting up the github.io repository

In this section you set up another repository, named `username.github.io`, for
the static content generated by Hugo. [GitHub Pages](https://pages.github.com/)
deploys the repository automatically to the site located at
https://username.github.io/.

#### Creating the repository

Let's start by creating the repository locally:

```sh
mkdir username.github.io
cd username.github.io
echo "# username.github.io" > README.md

git init
git add .
git commit -m "Initial commit"
```

Note that you created the repository with an initial commit. An empty repository
cannot be added as a git submodule, which is what you are about to do in a
second.

#### Publishing the repository

Publish the repository to GitHub using the `hub` command-line tool:

```sh
hub create
git push origin master
```

If you browse to https://username.github.io/ at this point, you will see that
the site is already live, using the contents of `README.md`. This is going to be
replaced by Hugo-generated content as you finish this walkthrough.

#### Cleaning up

You can now safely remove your local clone of `username.github.io`. You won't
need it anymore.

```sh
cd ..
rm -rf username.github.io
```

#### Linking the repositories

You are almost done with the repository setup. The final step is to link the
`username.github.io` repository to the `blog` repository, by making the former a
_git submodule_ of the latter.

Return to the `blog` repository, and invoke the following commands in
its top-level directory:

```sh
cd blog
git submodule add \
    https://github.com/username/username.github.io.git \
    public
git commit -am "Add submodule public"
```

The `public` directory is where Hugo generates the content. Adding the
repository as a submodule at this exact location makes it easy to push
the generated content to `username.github.io`.

## Continuous Deployment

We can now start to think about Continuous Deployment. Deploying a
change such as a new post to the live blog requires several steps:

1. You push the change to the `blog` repository.
2. Hugo is triggered to rebuild the site content.
3. The content is pushed to the `username.github.io` repository.
4. The repository is deployed to GitHub Pages.

In this section you set up continuous integration on the `blog` repository to
achieve steps 2 and 3, using [Travis CI](https://travis-ci.com). The last
step---deploying from `username.github.io` to GitHub Pages---does not require
further setup.

#### Setting up a bot account

Travis CI needs write access to the `username.github.io` repository to be able
to push the generated content to it. Instead of granting the CI job access to
your personal GitHub account, and thus to all of your repositories, you will set
up a separate bot account with collaborator access to the repository.

Create a GitHub account named `username-blog-bot`, replacing
`username` by your GitHub username. This can be done using GitHub's
[SignUp](https://github.com/join) page, after logging out of your
personal account. The bot account is just a normal GitHub user
account.

Note that you need to use a separate email address for the bot account, since
GitHub accounts must have unique email addresses. A useful technique in this
scenario is
[subaddressing](https://en.wikipedia.org/wiki/Email_address#Subaddressing) (also
known as _plus addressing_): Append `+blog-bot` to the local part of your email
address (the part before the `@` sign), and mails to that address will be
delivered to your normal inbox.

When you're done setting up the GitHub account, go to the _Settings_
page for the `username.github.io` repository, and add the bot account
as a collaborator.

#### Adding GitHub credentials to Travis CI

With the bot account set up, you can add the credentials to Travis CI.

On Travis CI, go to the _Settings_ page of the `blog` repository.

Add an environment variable named `GITHUB_AUTH_SECRET`.

Set the value to `https://user:pass@github.com`, using the credentials
of the newly created bot account.

Ensure that the _Display value in build log_ switch remains in the
_off_ position.

#### Configuring Travis CI

Travis CI is configured by adding a YAML configuration file named `.travis.yml`
to the top-level directory of the repository.

Continuous integration for the `blog` repository needs to perform three tasks:

1. Install Hugo into the CI environment.
2. Invoke the Hugo command-line tool to rebuild the site.
3. Deploy the new content to `username.github.io`.

The third step is delegated to a shell script, which is the subject of the next
section.

Create the file `.travis.yml` with the following contents:

```yaml
---
install:
  - curl -LO https://github.com/gohugoio/hugo/releases/download/v0.55.4/hugo_0.55.4_Linux-64bit.deb
  - sudo dpkg -i hugo_0.55.4_Linux-64bit.deb

script:
  - hugo

deploy:
  - provider: script
    script: ./deploy.sh
    skip_cleanup: true
    on:
      branch: master
```

Note that `skip_cleanup: true` is required so that Travis does not remove the
generated files before running the deployment script.

#### Adding the deployment script

Create the script `deploy.sh` in the `blog` repository, again replacing
`username` with your GitHub username:

```sh
#!/bin/bash

echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

cd public

if [ -n "$GITHUB_AUTH_SECRET" ]
then
    touch ~/.git-credentials
    chmod 0600 ~/.git-credentials
    echo $GITHUB_AUTH_SECRET > ~/.git-credentials

    git config credential.helper store
    git config user.email "username-blog-bot@users.noreply.github.com"
    git config user.name "username-blog-bot"
fi

git add .
git commit -m "Rebuild site"
git push --force origin HEAD:master
```

You can also invoke this script manually on your machine, after running `hugo`
to rebuild the site. Outside of CI, the script uses your normal GitHub
credentials to commit and push the generated content.

#### Finishing

Finally, commit the added files and push them to the `blog` repository.

```sh
git add .travis.yml deploy.sh
git commit -am "CI: Build and push to username.github.io"
git push
```

You can now visit https://travis-ci.com/USERNAME/blog to see your blog building.
When CI has completed, your blog should be live at https://username.github.io/.

#### Some remarks about the CI setup

Two remarks about the CI setup.

First, note that CI never updates the `blog` repository to point to the new
commit in the submodule. It cannot, because the bot account does not have write
access to this repository. This means that the `blog` repository is left
pointing at the initial commit of the submodule.

This is not really an issue, because our site is deployed directly from
`username.github.io`, rather than from the `blog` repository's submodule.

Second, note that the deployment script force-pushes the generated content,
effectively replacing `HEAD` and effacing history. This is no big deal, as the
repository only contains generated content.

The reason for force-pushing is somewhat subtle: As mentioned above, the
submodule still points to the initial commit in the `username.github.io`
repository. To perform a normal push you would therefore first need to pull from
origin. 

Unfortunately, this is impossible because the submodule is checked out in
detached `HEAD` mode and has no information about local and upstream branches.
Incidentally, this is also the reason why the last argument to the push command
is `HEAD:master` rather than `master`.

## Writing a post

Blog posts are written using
[Markdown](https://gohugo.io/content-management/formats/) syntax, with a
[YAML](https://gohugo.io/content-management/front-matter/) preamble called
_front matter_.

Invoke the Hugo command-line tool to generate the source file for the new post:

```sh
hugo new posts/my-first-post.md
```

The generated file is located at `content/posts/my-first-post.md` and
looks something like this:

```yaml
---
title: "My First Post"
date: 2019-04-27T10:48:29+02:00
draft: true
---

```

Use your favorite editor to write the actual post, and view your
changes locally using Hugo's built-in server:

```
hugo server --watch --buildDrafts
```

Remove the `draft` line from the front matter when the post is ready
to be published.

Commit and push. Your new post should go live once CI has completed.

## Links
- [Quick Start](https://gohugo.io/getting-started/quick-start/), from the official Hugo site
- [Host on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/), from the official Hugo site
- [Hugo on GitHub Pages with Travis CI](https://www.sidorenko.io/post/2018/12/hugo-on-github-pages-with-travis-ci/), by Artem Sidorenko