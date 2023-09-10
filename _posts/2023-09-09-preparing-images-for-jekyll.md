---
layout: post
title: "Preparing Images for Jekyll"
date: 2023-09-09 20:01
categories: Software
tags: [jekyll, lqip, gimp]
---

In playing with Jekyll and static sites, I've recently learned about the `webp` image format and LQIP (low quality image place-holders). At some point I'll make this better - but for now it is just some rough notes.

## Prerequisites

* ImageMagick (for the `convert` tool)
* <https://packages.debian.org/search?keywords=webp> (for the `cwebp` tool)


## Resize, Strip Metadata, and Convert to WEBP

A single file

```console
file="my-file.jpg"
convert "$file" -resize "1600x1600>" -strip "$file"
cwebp "$file" -o "${file%.*}.webp"
```

All JPG in the current directory

```bash
for file in *.jpg
do
    # note the use of the > operator
    # this will prevent smaller images from being scaled up
    convert "$file" -resize "1600x1600>" -strip "$file"
    cwebp "$file" -o "${file%.*}.webp"
done
```


## Generating LQIP

Using my CLI tool [lqip-gen](https://www.npmjs.com/package/lqip-gen) (post coming soon)

```bash
npm install -g lqip-gen

lqip-gen --copy test.jpg
```


## Chirpy Header Images

The [Chirpy theme](https://github.com/cotes2020/jekyll-theme-chirpy) for Jekyll recommends header images be `1200 x 630` or have an aspect ratio of `1.91 : 1`. I use [Gimp](https://www.gimp.org/) 