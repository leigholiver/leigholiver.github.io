---
title: Hello World (or, how the sausage is made)
date: 2021-05-23T12:11:07Z
tags:
  - github
  - gitops
  - hugo
---

I've been fascinated by the simplicity of static site generators recently, so when I set out to rebuild my personal website it seemed like a great opportunity to try them out. There are a lot of budget-friendly options for hosting this kind of static content, and the workflow of building and deploying the site each time content is added is ideal for setting up some automation.

<!--more-->

I use [GitHub](https://github.com) for version control and they have two services for exactly this use-case - [GitHub Pages](https://pages.github.com/) which allows you to host a website from a git repo, and [GitHub Actions](https://github.com/features/actions) which provides a way to automate the build and deploy process. Deploying a site like this is wonderfully straightforward:

1. Build the site content
2. Commit the result to the `gh-pages` branch
3. The site is ready to go at `<username>.github.io`

Result! But of course, first I needed a site.

GitHub Pages has good support for [Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll) out of the box, but I decided to use [Hugo](https://gohugo.io/). Hugo has [really fast build times](https://css-tricks.com/comparing-static-site-generator-build-times/) which scale well as more content is added - I've got  to make the most of my free GitHub Actions minutes! Initially I picked out a theme from their [massive themes library](https://themes.gohugo.io/), but found that the themes I liked tended to have slower load times than the basic Jekyll theme I had been testing. Continuing the theme of *simplicity*, I ended up creating [my own theme](https://github.com/leigholiver/bulb) using the [Bulma CSS framework](https://bulma.io/), which is really light.

The last job was to set up the automation with GitHub Actions so that when I push a commit to the `master` branch with a new post, the site gets updated automatically. Hugo and GitHub pages are both quite popular in the GitHub Actions marketplace, and I used [`peaceiris/actions-hugo`](https://github.com/peaceiris/actions-hugo) to set up Hugo and [`peaceiris/actions-gh-pages`](https://github.com/peaceiris/actions-gh-pages) to push the resulting files to the `gh-pages` branch. I also added a step to create a `CNAME` file, which allows me to use my own domain name rather than `<username>.github.io`.

All in all I'm very pleased with how straightforward this was to set up. The code for this site and the Actions workflow can be found on [my GitHub](https://github.com/leigholiver/leigholiver.github.io).
