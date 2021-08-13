+++
title = "We built this website in 3 hours!"
date = "2021-08-11T14:59:44+01:00"
author = "Alex G"
authorTwitter = "" #do not include @
tags = ["hugo", "simplify"]
keywords = ["hugo", ""]
description = "How we built this website and went live in 3 hours"
showFullContent = false
cover = "img/hugo-logo-wide.svg"
+++

As you guessed it was using [**Hugo**](https://gohugo.io/), a framework for building websites.

## Our website requirements

#### Development

- Zero HTML,CSS and JS coding.
- Easy CI & CD (Especially the deployment)
- Git integrated.
- Static site for simplicity, security & speed.
- Easy to add content.

#### Hosting

- As simple and affordable as possible.
- Built in TSL (SSL).
- Safe, reliable & fast.

With this list we put out the question and [**Hugo**](https://gohugo.io/) was somewhere in the mix.

We watched one video on YouTube about Hugo!

{{< youtube id="qtIqKaDlqXo" title="Introduction to Hugo | Hugo - Static Site Generator | Tutorial 1" >}}

Had the site running in 20 minutes using the theme [**Terminal**](https://github.com/panr/hugo-theme-terminal) by [**panr**](https://github.com/panr)

`Tip`: Have a look at the Terminal theme sample site in the `content` and `content\blogs` folders to see how they did their Markdown and just copy that.

### Where to host

We did not like the the option using GitHub Pages so went for [**Netlify**](https://www.netlify.com/) as we thought it was just a beautiful piece of technology to work with and so simple to use.

Our workflow:

- Commit content to a git `dev` branch.
- Merge to `main` branch when complete.
- Push to Github.
- Netlify auto detects new changes on `main` and deploys the site.
- Hugo auto detects changes and updates before your eyes.

It's magic!