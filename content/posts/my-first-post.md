+++
title = "How we built this website in 3 hours"
date = "2021-08-11T14:59:44+01:00"
author = "Alex G"
authorTwitter = "" #do not include @
cover = ""
tags = ["hugo", "simplify"]
keywords = ["hugo", ""]
description = "How to build a website and go live in 3 hours"
showFullContent = false
+++

Spoiler alert, we did it using [**Hugo**](https://gohugo.io/)

## Our goals were

- Minimmal to zero HTML,CSS and JS coding.
- Simple deployment. No dedicated CI tools.
- Integrated with git using Github or Bitbucket.
- No server or ssl management.
- Static site generator for simplicity, security & speed.
- Easy to update and roll back.

## Hosting requirements

- As easy and cheap as possible.
- Safe & reliable.

Using this list we put out the question to our developer friends and [**Hugo**](https://gohugo.io/) was somewhere in the mix.

We watched one video on YouTube!

[![The Simple Engineer](/img/hugo-in-10-mins.jpeg)](https://www.youtube.com/watch?v=qtIqKaDlqXo&list=PLLAZ4kZ9dFpOnyRlyS-liKL5ReHDcj4G3)

Had the site running in 20 minutes using the theme [**Terminal**](https://github.com/panr/hugo-theme-terminal) by [**panr**](https://github.com/panr)

### Hosting

We did not like the the option using GitHub Pages so went for [**Netlify**](https://www.netlify.com/) as we thought it was just a beautiful bit of technology to work with.

`Tip`: Have a look at the Terminal theme sample site in the `content` and `content\blogs` folders to see how they did their Markdown and just copy that.

Our workflow:

- Commit content to a git dev branch.
- Merge to main branch when complete.
- Push to Github.
- Netlify auto detects new changes on master and deploys.

It couldn't be more simple.

Enjoy!
