---
layout: post
title: "Static Website with Jekyll and Chirpy"
date: 2023-09-04 11:23
categories: Software
tags: [jekyll, documentation, static website, github]
---

Up until this point, my notes and documentation have been all over the place. Originally, I was self-hosting a Gitea instance (before GitHub was giving out unlimited private repos) and storing `README.md` alongside projects. But I quickly realized the issues with this approach when I needed some notes when my services were down. Next, I've kept a series of markdown notes in a folder synced with my Nextcloud instance. I've toyed with the idea of starting a Wordpress site, but have decided to give Jekyll a try.

1. It is hosted outside of my infrastructure
1. I'm already writing most of my notes in markdown
1. I've played with Lektor some, but wanted to learn something new


## Setting up Ruby

I decided to use [rbenv](https://github.com/rbenv/rbenv) as I use similar tools to manage other environments (e.g. [tfenv](https://github.com/tfutils/tfenv), [nvm](https://github.com/nvm-sh/nvm), etc.)

I'm running Debian on my machine.

```bash
# install dependencies
sudo apt-get install autoconf bison build-essential libssl-dev libyaml-dev libreadline6-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm3 libgdbm-dev

# install git
sudo apt-get install git-core

# clone rbenv and add it to path
git clone https://github.com/rbenv/rbenv.git ~/.rbenv

echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc

source ~/.bashrc

# add the ruby-build plugin
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

# list out available versions
rbenv install -l

# install a version of ruby
rbenv install 3.2.2

# set global default version
rbenv global 3.2.2
```


## Choosing a Theme

There are many themes for Jekyll to choose from, but in the end I selected [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy). My main considerations were:

1. Search
1. Tags and categories
1. Minimal and lightweight
1. Includes GitHub actions to build and publish the site to GitHub pages
1. The timeline is cool
1. Provides a GitHub [starter project](https://github.com/cotes2020/chirpy-starter) to easily get started


## Getting Started

1. Navigate to the starter project (<https://github.com/cotes2020/chirpy-starter>) and click `Use this template > Create a new repository`

1. The repository name should match your GitHub username (if you are planning on using GitHub to host the site)

1. Clone the repository

1. Install the ruby dependencies

    ```bash'
    bundle
    ```

1. Review and modify `_config.yml` (documentation link: <https://chirpy.cotes.page/posts/getting-started/>)

1. Create a first post markdown file (e.g. `_posts/2023-09-04-my-first-post.md`)

1. Start the local server and review the page live

    ```bash
    bundle exec jekyll s
    ```

1. Navigate to http://localhost:4000 to view the site


## Verify Custom Domain for GitHub Pages (Recommended)

The following steps can increase the security of your custom domain and avoid takeover attacks.

Documentation link: <https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages>

1. Login to GitHub

1. Click on profile picture > Settings

1. Select `Pages` menu item on left

1. Add the domain and create the necessary TXT records


## Create CNAME (optional)

The following step is only required if you are using a custom domain name (`https://docs.samholton.com`) rather than the GitHub Pages domain (`https://samholton.github.io`)

Documentation link: <https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site>

I'm using a sub-domain rather than the APEX domain due to other self-hosted services behind my domain. I configure my DNS for `*.samholton.com` to route to my k8s ingress controller so I can easily expose new services without explicitly enumerating them in DNS.

1. Create CNAME for `docs.samholton.com` pointing to `samholton.github.io`


## Enable GitHub Pages

1. Navigate to the settings of you new repo (<https://github.com/samholton/samholton/settings/pages>)

1. Click `Pages` on the left navigation

1. Under `Source` select `GitHub Actions`

1. Add the custom domain (`docs.samholton.com`)


## Commit, Push, and Build

The [chirpy starter](https://github.com/cotes2020/chirpy-starter) includes a GitHub action (`.github/pages-deploy.yml`). So all that is needed to publish to GitHub pages is to push to the `main` branch (as defined in `.github/pages`)

```yaml
name: "Build and Deploy"
on:
  push:
    branches:
      - main
      - master
```
