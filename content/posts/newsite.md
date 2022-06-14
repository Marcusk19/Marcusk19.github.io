---
title: "Migrating To a New Site"
date: 2022-06-14T09:28:51-04:00
draft: false
---



Wordpress has been excellent for getting my site up and running. It makes it easy to change the look, theme, and content of your site. However, it comes with a few inconveniences that I have noticed during these last months:

* Wordpress comes with a lot of bloat. Things like SEO optimization and OptinMonster, which may be relevant to other people but not for my uses.

* Forms that allowed comments underneath my posts led to a few bots spamming me.

* Themes did not look as clean as I would like them to be.

While some of these things could probably be changed through me just getting more familiar with Wordpress, I wanted to move towards a simpler way of maintaining my website. Ideally, I wanted to write posts on my local machine, and then pushed them to my website when ready. 



I found this open source project called Hugo, which allows you to write files using markdown syntax and converts them to HTML/CSS format to serve as a static site. This was perfect for me, because when boiled down to its simplest form my website's only purpose was to serve a couple pages. In other words, there was no need for me to stick with Wordpress features that were more suitable for online retail sites, which have more complexity than what I was using my site for.



## Getting Started

Setting up Hugo to serve my websites was very frictionless.  Following their [quickstart](https://gohugo.io/getting-started/quick-start/) guide had me up and running in only 10 minutes. I had to convert my old posts online to markdown, but that was made easy through a convenient browser extension that downloads webpages to a .md file: [MarkDownload](https://chrome.google.com/webstore/detail/markdownload-markdown-web/pcmpcfapbekmbjjkdalcgopdkipoggdi?hl=en-GB). 



I did have to add my own metadata at the top for my old posts. Typically you would run `hugo new posts/my-post.md` and the relevant data would be filled in at the top automatically. However, the downloaded files were missing that header.

```markdown
---
title: "Single GPU passthrough with AMD build"
date: 2022-06-08T21:57:09+00:00
draft: false
---
```



Setting `draft: false` lets Hugo know to pick up that .md file and serve it when running something like `hugo server` . Alternatively, passing the -D flag will make hugo serve all draft posts like so: `hugo server -D`. 



## Making it Look Good

Getting a good theme for my site was a big reason why I wanted to switch. There's a lot of great themes that can be found [here](https://themes.gohugo.io/). I went with [m10c](https://themes.gohugo.io/themes/hugo-theme-m10c/) for its simplicity and color scheme. 



 `git submodule add https://github.com/vaga/hugo-theme-m10c.git themes/m10c` created a `m10c` folder in my `themes` directory. Further down the line I ran into issues with the theme while pushing to GitHub. It complained about the `example_site` directory, and my workaround was to fork my own copy of the repo and remove `example_site`. Then I added it to my project under `themes/my_m10c`. 



I only took a look at this theme, but suspect that the config varies for other ones. Editing the `config.toml` file is necessary to get your theme to show up.

Defining theme and menu items (Note: your baseURL should match whatever domain you want pointing to your site):

```toml
baseURL = 'https://marcusk19.github.io/'
languageCode = 'en-us'
title = 'Marcus Kok'
theme = "my_m10c"

[menu]
    [[menu.main]]
        identifier = "home"
        name = "Posts"
        url = "/"
        weight = 1
    [[menu.main]]
        identifier = "about"
        name = "About"
        url = "/about/"
        weight = 1
    [[menu.main]]
        identifier = "resume"
        name = "CV"
        url = "/resume/"
        weight = 1
```

 To add icons linking to my GitHub and LinkedIn profiles:

```toml
[params]
    author = 'MarcusK'
    description = 'I like to code and break things'
    menu_item_separator = '|'
    [[params.social]]
        icon = "github"
        name = "My Github"
        url = "https://github.com/Marcusk19"
    [[params.social]]
        icon = "linkedin"
        name = "LinkedIn"
        url = "https://www.linkedin.com/in/marcus-kok-9a80551a3/"
```



To change the avatar icon and favicon, I added the images I wanted in the `static` directory named `avatar.jpg` and `favicon.ico` respectively.



## Hosting on GitHub

I chose to use GitHub pages to host my site because it's free and I can use a custom url for it. According to the guides, I made a `gh-pages.yml` file and placed it in `.github/workflows` found in the root directory of my project:

```yml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

Once that was done, I committed my changes, ran `hugo` to build the static files, and pushed it to my repo. GitHub ran a pipeline for my site to get built and deployed. From there I only had to set a custom domain name for it.



I got a free domain through Hostinger, which was where I got my previous *marcuskok.tech* domain. Because I didn't want to take down my old site yet, I obtained *marcuskok.com* free for one year. It took a little fiddling to get it pointing to my new site. What I did was create A records for my domain through Hostinger's management panel that pointed to GitHub's IP addresses. More information can be found [here](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain). 



Once DNS propagation finished, I now have my new site up at *marcuskok.com*. I will probably keep the old one around until my free domain ends, at which point I'll move whichever name is cheaper to my main site. Overall, I greatly prefer the simplicity and look of the new site, and am pleased with how easy Hugo makes it to build a website.
