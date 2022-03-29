---
layout: post
title: "Jekyll: Dev'ing elsewhere & Publishing on GitHub Pages"
date: 2019-08-25 19:37
last_modified_at: 2022-03-29 18:26
---

This website is hosted and published using GitHub Pages but I do all the dev 
work on another server.

Having been the president of Swansea University Computer Society (SUCS) for 2 
years whilst at university I was ~~cruelly punished~~ rewarded with lifetime 
membership, this means I have access to society resources such as a web 
hosting environment. It's a diverse shared environment with many users, all of 
which don't have root so libraries and such need to be installed per-user.

GitHub Pages is a great way of hosting a website for free, especially if it's 
small and static, you can even point your own domain at it.

## GitHub

Behind the scenes GitHub Pages uses Jekyll, even if you don't.

On GitHub I have a standard setup, a repo called `<username>.github.io` with a 
custom domain setup to point to it.

To get the basics of a site going we need a couple of files in this repo:

- `_config.yml` - Jekyll main config file

```
# Site settings
title: Imran's Website
description: >-
  Full(-ish) of thoughts, musings, 'stuff'.
baseurl: "/"
url: "https://imranh.co.uk"
show_excerpts: true
twitter_username: imranh2
github_username:  imranh2
  
# Build settings
markdown: kramdown
theme: minima
```

- `index.md` - Simple file telling Jekyll's minima theme to give us a homepage

```
---
layout: home
---
```

These 2 simple files will give you a working website in no time at all, but 
what about development? Every minor tweak you have to push to your live 
website potentially causing breakages in production and ending up with a 
inflated git commit history.

## SUCS (or another web-host with shell access and Ruby)

So if you're a SUCS member this will work for you. If you're using another 
web-host your millage may vary.

Once SSH'd into your host (silver/sucs.org for SUCS members) you'll want to 
navigate to your public_html folder (wherever your webroot/docroot is).

`$ mkdir ~/public_html; cd ~/public_html`

Once you're there you are going to want to clone your GitHub repo down:

`$ git clone https://github.com/<username>/<username>.github.io`

Then we need to go into the cloned folder and do a few things.

`$ cd <username>.github.io`

Create a file called `Gemfile` with the following contents:

```
source "https://rubygems.org"
gem "github-pages", group: :jekyll_plugins
```

Install the magic that makes GitHub Pages work on a local machine:

`$ bundle install --path vendor/bundle`

Edit your `_config.yml` file and update the `baseurl` and `url` to match your 
environment, mine looks like:

```
baseurl: "/~imranh/imranh2.github.io/_site"
url: "https://sucs.org"
```

And then finally build your site by doing:

`$ bundle exec jekyll build`

In SUCS you can then go to `https://sucs.org/~<username>/<username>.github.io/_site` 
to view your site, it should be identical to what's hosted at `https://<username>.github.io`

Now you can just edit the files as you wish, running `bundle exec jekyll build` 
each time you want to preview your changes, once your happy with a change add 
the files to a commit and push them up to GitHub for the world to enjoy them.

## Notes & Warnings

Changing the `baseurl` and `url` in the `_config.yml` and committing that to 
your repo will cause things to break, so avoid pushing it up to GitHub.

My `.gitignore` file also looks like:

```
*.lock
.bundle/
vendor/
_site/
.sass-cache/
.jekyll-cache/
.jekyll-metadata
```

As to avoid committing files that don't need to be tracked.