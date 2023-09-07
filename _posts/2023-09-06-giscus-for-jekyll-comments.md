---
layout: post
title: "Giscus and Jekyll - Comments on Static Sites"
date: 2023-09-06 21:36
categories: Software
tags: [jekyll, static website, github]

image:
  alt: comment form
  path: /assets/images/headers/giscus-comments.webp
  lqip: data:image/webp;base64,UklGRiYAAABXRUJQVlA4IBoAAAAwAQCdASoQAAgABUB8JaQAA3AA/vB8JAAAAA==
---

Originally, I was looking at using [staticman](https://github.com/eduardoboucas/staticman) to add comment functionality. This is a NodeJS app you host to receive `POST` requests from forms on your static site. With this, it creates PRs against your docs repo to add the comment content. Optional moderation is in the form of approving the PR (vs. auto merging the PR).

## Giscus

But then I stumbled across [giscus](https://github.com/giscus/giscus) when looking at the configuration options for the [chirpy theme](https://github.com/cotes2020/jekyll-theme-chirpy) that I'm using. The documentation is great, and the process is very smooth, so my notes are minimal.

Keep in mind that this uses GitHub Discussions. So commenters must have a GitHub account. To make comments from the static site itself, they must also enable the `giscus` bot. Otherwise, they can use GitHub Discussions directly.

## Getting Started

After reviewing the Chirpy config, the only two items I wasn't initially sure how to obtain were `repo_id` and `category_id`. However, the wizard on <https://giscus.app/> provides all of the necessary values.

1. Enable `Discussions` on your static website repo (click the `Settings` icon on your repo and enable the `Discussions` checkbox under `Features`)

1. Edit the discussion categories on your repo by clicking the edit/pencil icon next to `Categories` on the left of the `Discussions` tab (e.g. <https://github.com/samholton/samholton/discussions>)

1. I created a new category called `Comments`

1. Install the [giscus app](https://github.com/apps/giscus) - only give it permission to your static website repo

    ![giscus app install](/assets/images/posts/giscus-app-install.webp){: width="400" height="400" lqip="data:image/webp;base64,UklGRioAAABXRUJQVlA4IB4AAAAwAQCdASoNABAABUB8JaQAA3AA/vBM/YRYR+zAAAA="}

1. Complete the wizard on <https://giscus.app/>

    * Enter your repository name (`samholton/samholton`)
    * Select your category (`Comments`)
    * Customize any other option(s)
    * Retrieve the necessary configuration for Chirpy under the `Enable giscus` section.

1. Populate the `giscus` values in `_config.yml`
