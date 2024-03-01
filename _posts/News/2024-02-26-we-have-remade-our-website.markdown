---
layout: post
title:  "We have moved to a new layout and site!"
date:   2024-02-26 16:40:51 +0800
categories: [General, News]
tags: News
author: Tivin
---
If you've been to the site more than once you may have noticed that the website changed look and feel changed.

I like experimenting with "new" (new to me) technology and ways to make life easier, this one is using [Jekyll] and [Front Matter] to generate a static website.

## Why a SSG?
This blog (my wiki and what I do) is a very static site, a CMS like ghost where I need to manage a database and a few other components is not something that is actually needed for a site that gets updated with a few posts once a while.
Another reason is security. It is much easier to protect a static site rather than a dynamic one.

For the reason above, I decided to try and use a Static Site Generator (SSG) and hopefully I decide to stick to it as migrating the content takes quite a bit of time.

This is also going to speed up the loading of the website as there is no more a database connection that is needed in the backend, and over time I will make a pipeline to automate the deployment of the site across the 3 webservers that are maintained for various geographies.

Hope ya'll enjoy!

[Jekyll]: https://github.com/jekyll/jekyll
[Front Matter]: https://frontmatter.codes/docs/markdown
