---
title: How to create this page
published: true
---
Guide of how to create your web site.
## [](#header-2)First steps
First step is create your **own acount** on [GitHub](https://github.com/). Then visit the style you want to use and
_(Remeber to do Fork of repository to help the comunity)_
* [Hacker-Blog](http://github.com/tocttou/hacker-blog).
* [Neumorphism](https://github.com/longpdo/neumorphism).

## [](#header-2)Second steps
Second step is change the name of the repository: Go to the `settings` and then give the name you want ending with `<your-name>.github.io`

## [](#header-3)More of usage

### [](header-4)Local Build
If you want to see the changes before puching the blog to GitHub, do a local build.
1. `gem install jekyll`
1. `gem install jekyll-seo-tag`
1. `gem install jekyll-paginte`
1. `gem install jekyl-sitemap`
1. (`cd` to the blog directory, then:) `jekyll serve --watch --port 8000`
1. Go to `https://localhost:8000/` in your web browser.

### [](header-5)Update your local data
When you have all the local change finished, you can update your web following this steps:
1. `git add .`
1. `git commit -m "<name-of-update>"` This show you the change you will update.
1. `git push -u origin` This will require your name user and your token-password. [Crear token-password](https://docs.github.com/es/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
