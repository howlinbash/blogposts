---
layout: post
title: How I Built howlinbash.com
---

TL/DR: I built howlinbash.com the *long* way.

I needed a blog.

I didn't spend too much time shopping before I settled with a self-hosted implementation of Jekyll. I'm a bit of a command line junkie. The draw of writing, formating and publishing each blogpost without having to touch a mouse was too enticing to pass.

The next step was to plan out a modular development structure for the website.

I decided to split the code into three repositories: blogposts, website and website theme. By isolating the blogposts, I could compensate my lack of a GUI by drafting posts from [github.com](https://github.com) if I needed to. By seperating the theme from the main site, I could switch to a new theme without having to refactor the entire codebase and I could potentially use either repo for future websites. I chose [Hyde](http://hyde.getpoole.com/) by [@mdo](https://twitter.com/mdo) as a base theme. Forked it, updated it and called it [Heidi](https://github.com/howlinbash/heidi).

Commiting to this model, the challenge now was to figure out exactly what each repo would look like, how to develop each side-by-side and how to bring them together as the finished site. 

After a little research, I decided to turn the blogpost repo into a submodule of the main site repo. A simple `git submodule update` command would fetch the latest posts. However, the submodule approach proved too clumsy for the theme repo, so I decided instead to develop Heidi as a seperate ruby gem.


## The Server

I had a number of requirements from my server:

1. TLS encryption ( https:// instead of http:// )
2. A staging domain to make sure the site works in a live environment
3. A reverse-proxy to easy facilitate future capabilities
4. A docker-centric workflow ( my latest infatuation )

I began developing the most excessive, scalable, docker server cluster I could imagine; With load balancing, continuous deployment, failover, all spread across as many virtual servers as I desired. By the time I was knee deep in iptables documentation, I realised I'd gone too far. Vowing to return one day to my mega cluster, I pushed on, building a simple NGiNX reverse-proxy server before finding a superb [pre-built solution](https://github.com/howlinbash/build-server) on GitHub.

The server now consists of five parts

- The main NGiNX reverse-proxy container
- A container that generates and updates the reverse-proxy configuration
- A container that handles the acquisition and renewal of the TLS encryption certificates
- A container with the staging website available at https://dev.howlinbash.com
- And finally a container with the live website: https://howlinbash.com


## The Development Environment

To build a seperate theme whilst building a website turned out to be my most disruptive decision. My most puzzling problem to solve.
- Which repo should I commit which code changes to?
- How can I make sure the changes I make to the theme will load on the site?

To clarify:

My goal was to build a website, it was not to build a theme. The theme would be a by-product of the website. Ideally as I built the website, the theme would magically build itself behind the scenes, without me having to think about it. This meant that I would need to develop the website from both the theme repo and the website repo almost simultaneously.

There were no two ways about it. I needed a bash script.


## The Build Script

The [script](https://github.com/howlinbash/howlinbash/tree/master/build-scripts) is run from `main.sh` ( which can be aliased to whatever you like ).

It takes a number of options initialised by a simple case statement and two of those options also take a further option, again determined by a case statement.

Below is a description of what each option does and how it works.

### Load
> Move posts, pages & config from site repo to theme repo for development.

Although the code I commit will be pushed to my theme repo rather than my site repo, by initially loading the files and configuration specific to my website, my theme server will appear as if i'm working on my website code directly.

- `sed` comments out the howlinbash specific settings in the config file
- `git update-index --assume-unchanged` ignores the howlinbash specific files whilst developing

### Serve
> Serve and watch theme or site or preview blogpost.

The serve option itself has four options
- `theme` for theme focused development
- `site` for site focused development
- `post` to preview a draft blogpost
- `stop` to stop a running site server

If I serve my theme, the script moves to the theme repo and runs `bundle exec guard` to serve a live-reloading website to localhost.

However, if I serve my site, the script 
- moves to the site repo
- starts a docker container that watches and builds the site to `/web`
- starts a second container that watches and serves the same directory to localhost

To preview blogposts I decided to re-appropriate gits functionality to the needs of the script.
- the blogposts repo has 2 remote branches: `master` and `drafts`
- each blogpost has it's own local branch that is merged with `drafts` each time a post is previewed

The script
- starts the above site server
- commits the post to the local blogpost branch
- merges the the blogpost branch to `drafts` which is then pushed to GitHub
- switches to the site repo (which contains the blogposts repo as a submodule)
- switches the blogposts submodule branch from `master` to `drafts`
- pulls the latest draft blogpost
- the site server, started above, spots the change and rebuilds the site for preview

This process is resolved when a blogpost is posted to the website with the `post` option, detailed below.

The `stop` option simply stops and removes the containers that are serving the site locally.

### Test
> Test theme gem by pushing, pulling and building from local gem server.

To avoid commiting broken code I have to be able to test my theme works when loaded as a gem.

The script 
- loads any changes from the theme config file to the site config file
- starts a docker container with a local [geminabox](https://github.com/geminabox/geminabox) gem server
- switches the theme settings to test theme settings in the config file
- makes a test gem and pushes it to the geminabox server
- serves the test site in a docker container to localhost
- reverts all config settings back to the original theme settings

### Bump
> Update theme version number, commit changes and post gem to rubygems.org.

When I know that the theme works as a gem and I'm happy with my changes, I bump, commit and publish.

The script
- makes a gem
- bumps the version number in the relevant config files
- pushes the gem to [rubygems](https://rubygems.org/gems/jekyll-theme-heidi)
- commits the changes to GitHub
- interactively commits the config file to commit individual sections if needed
- tags the commit with the version number

### Deploy
> Deploy code to howlinbash.com

The deploy option itself has three options
- `stage` pushes the latest code to staging @ dev.howlinbash.com
- `live` switches the live code ( @ howlinbash.com ) with the staged code
- `revert` reverses the switch just in case some bad code slips through

The deploy system relies mainly on docker tags to move and switch docker images.

The tags are:
- `next` this is the latest image. The latest code written
- `current` this is the code you can currently see at howlinbash.com
- `previous` the last website that was live

While the above images live on [docker hub](https://hub.docker.com/r/howlinbash/howlinbash) and are pushed and pulled from my local machine to my server, the next two images only live on the server and determine what is viewable at dev.howlinbash.com and howlinbash.com
- `staging` the image viewable at dev.howlinbash.com
- `live` the image viewable at howlinbash.com

`stage`
- pulls the latest blogposts
- builds an image and tags it `next`
- pushes the image to docker hub
- SSHes into the server
- pulls the `next` image and retags it `staging`
- reboots the server to load image to dev.howlinbash.com

`live`
- switches the `current` image to `previous` on docker hub
- pushes the recently built `next` image to docker hub as `current`
- SSHes into the server
- retags the `staging` image as `live`
- pulls the `previous` image from docker hub as `staging`
- reloads the server to load images to dev.howlinbash.com and howlinbash.com

`revert`
- SSHes into the server
- switches the `staging` image with the `live` image on the server
- reloads the server
- switches the `previous` image with the `current` image on docker hub

![deploy diagram]({{ site.url }}/assets/img/deploy.jpg)

### Post
> Post blogpost to website.

Resolving the git appropriation from `serve post`, the script pushes the completed post to `master` deletes the draft from `drafts` and then merges `master` into `drafts` to keep `drafts` housing drafts and all complete posts and `master` housing just the complete posts.

The script
- reads the post filename from the user
- checksout blogposts `master`
- checksout the blogpost file from the blogpost branch
- commits to `master` and pushes `master` to GitHub
- deletes the file from `drafts` and then deletes the blogpost branch
- builds a `next` site image with the latest post from blogpost `master`
- switches `current` to `previous` on docker hub
- pushes `next` image to docker hub as `current`
- SSHes into server
- pulls `previous` as `staging` and `current` as `live`
- reloads server
