+++
title = "Obligatory first post: Setting up the blog"
date = 2025-04-03
taxonomies.categories = ["webdev"]
+++

# Hello world!

Finally decided on [Zola](https://www.getzola.org/) for the blog. I was inches away from writing my own SSG but that's too much yak shaving. For now...
Instead I'll share how easy and fast it was to get started with this blog.

## Installing zola
There are plenty options on Zola's website, but if you don't have any of the package managers, distros or some other issue like me, you can get the binaries from the [github repo](https://github.com/getzola/zola).

``` bash
# Grab the binaries
curl -L -o zola.tar.gz "https://github.com/getzola/zola/releases/download/v0.20.0/zola-v0.20.0-x86_64-unknown-linux-gnu.tar.gz"

# Unpack
tar -xvzf zola-v0.20.0-x86_64-unknown-linux-gnu.tar.gz

# Move it to make it available for all users and available in bash
sudo mv zola /usr/local/bin
``` 

Now Zola is installed.  

## Make your site

Now we are ready to make the site and install a theme (Or make your own)

``` bash
# Init the blog and follow the questions - Besides using syntax highlighting, I went with the default options
zola init myblog

# Run your site locally
zola serve
``` 

## Install a theme
Now you can grab a theme from Zola's site or anywhere else.
I started with the "Hook" theme by [Koen Bolhuis](https://koen.bolhu.is/).

``` bash
# Navigate to your themes folder
cd themes

# Grab the theme
git clone https://github.com/InputUsername/zola-hook.git hook
```

Put this in your `config.toml` at the top
``` toml
theme = "hook"
```
Now copy the templates from the theme which you can easily modify. Make sure you are in the root folder of the project.

```bash
cp -r themes/hook/content .
```

There you go - you now have a simple blog up and running locally.
You can run `zola build` to build the site into a `public/` folder that can be hosted statically on e.g. github or netlify. There's instructions in the docs for Zola on how to automate building the site for github pages everytime you push a commit too. Happy blogging 🍺