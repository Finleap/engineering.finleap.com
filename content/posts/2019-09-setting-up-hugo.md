---
title: "Setting Up Hugo"
date: 2019-09-16T10:05:13+02:00
draft: false
description: "Hugo is a Golang-based static site generator. It reads content created in Markdown format, applies a set of templates, and renders HTML files which can then be uploaded to a server"
---
Hugo is a Golang-based static site generator. It reads content created in Markdown format, applies a set of templates, and renders HTML files which can then be uploaded to a server.

*This is the first in a series of X posts covering the setup, configuration and deployment of a statically-generated HTML website using Hugo, Amazon Web Services, and GitLab. More [here]({{< ref "/posts/2019-09-blogging-platform" >}})*

The advantage of a static site generator over, say, a dynamic blogging platform such as Wordpress is threefold - firstly there are fewer moving parts (e.g. no database or PHP interpreter to deal with).  Secondly, serving HTML files is orders of magnitude faster than a system which requires database queries to be run before a page can be build. Finally, if the HTML content is static, it can be cached either server-side or in a content distribution network (CDN) which makes the site significantly more robust under high traffic conditions.

There's another advantage which is less obvious to end users, but useful for us engineers - the build-and-render process of a static site generator sites very well with continuous integration pipelines. This means that the whole process of creating, building and deploying the site can be largely automated and triggered with pull requests.

# Setting up Hugo

Setting up Hugo is a 5-stage process:

1. [Install the Hugo binaries](#installing-hugo)
1. [Create a site](#creating-a-new-site)
1. [Configure the site by installing templates etc](#configure-the-site)
1. [Creating some initial content](#create-some-initial-content)
1. [Building the site and running it locally](#build-and-run-the-site-locally)

## Installing Hugo

Hugo can be downloaded from [the Hugo site](https://gohugo.io/) - it's available for Mac, Linux and Windows. The [quickstart guide](https://gohugo.io/getting-started/quick-start/) has the full instructions.

## Creating a new site

When creating a new site, Hugo sets up a directory structure containing two parts - the Hugo files which are needed to configure the site (e.g. config files, templates etc) and the rendered HTML files which form the site itself.  It also ships with a local server, so you can run it up on your development machine and see the results in real time.

The command to create a new site is

`hugo new site <sitename>` where `<sitename>` will be the name of the directory that Hugo creates.

## Configure the site

Before Hugo can render a site, it needs a theme. There's a range available [on the Hugo site](https://themes.gohugo.io/), so there's no immediate need to start building one yourself.

To install a theme, you can add it as a Git submodule:

1. Switch to the `<sitename>` subdirectory
1. Initialise a Git project here: `git init`
1. Add the theme of your choice as a Git submodule:
`git submodule add <url_to_github_repo_of_theme> themes/<theme_name>`

This will install the theme's files in a subdirectory underneath the `themes` directory:
```
-- site root
 |- themes
     |- <theme_name>
```
Then you'll need to update the `config.toml` file to tell Hugo to use the new theme:
1. Open the `config.toml` file - this is located in the site root
1. Add the line `theme = <theme_name>` and save

## Create some initial content

Hugo uses Markdown-formatted files to render the viewable HTML pages.  The naming convention is relatively flexible, but for the purposes of getting started, we'll use a date-based format.

Create a new post with `hugo new posts/yyyy-mm-dd-my-first-post.md` where `yyyy-mm-dd`is the current date.

This will create a new Markdown file called `yyyy-mm-dd-my-first-post.md` in the `./posts` directory.

Open the file in a text editor, and you'll see a blank post with basic metadata at the top:

```
---
title: "2019 09 16 My First Post"
date: 2019-09-16T11:21:52+02:00
draft: true
---
```

Hugo makes a best-efforts guess at the title, as well as setting the draft status to `true`. We'll use this later to configure the display of draft posts on the staging site, but suppressing them on the live site.

Another useful metadata field is the `description` - some themes are configured to display this as a post slug. To add a post description, add it inside the metadata header:

```
---
title: "My First Post"
date: 2019-09-16T11:21:52+02:00
draft: true
description: This is the description of my first post
---
```

The post content should be added below the header in Markdown format. Markdown is beyond the scope of this post, but there are [numerous](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) [cheatsheets](https://www.makeuseof.com/tag/printable-markdown-cheat-sheet/) a short Google search away.

## Build and run the site locally

Hugo ships with a built-in web server that allows you to preview the site locally. It also hot-reloads the site, which means that changes will be reflected in the browser as soon as the Markdown file is saved.

The local server is started with

`hugo server -D`

The `-D` switch forces it to display posts with `draft` status. When it's started, the server will dump some diagnostic data to the console, and then expose the website on port 1313:

[`http://localhost:1313`](http://localhost:1313)

```
| EN
+------------------+----+
Pages            | 22
Paginator pages  |  0
Non-page files   |  0
Static files     |  4
Processed images |  0
Aliases          |  0
Sitemaps         |  1
Cleaned          |  0

Total in 16 ms
Watching for changes in /Users/finleap_user/Documents/code/finleap-tech-blog/{archetypes,content,layouts,themes}
Watching for config changes in /Users/finleap_user/Documents/code/finleap-tech-blog/config.toml, /Users/finleap_user/Documents/code/finleap-tech-blog/config/_default
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

With Hugo set up to generate site content, it's time to move onto the next step:
