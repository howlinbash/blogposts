---
layout: post
title: How I Built HowlinBash.com
---

Answer: I built HowlinBash.com the *long* way.

As I saw it there were 5 main approaches open to me.

1. Medium
2. Squarespace
3. WordPress
4. GitHub Pages
5. Other

I chose other.

My four main requirements were:

- workflow speed (of development and blogging)
- maintainability
- extendability
- an accurate showcase of my current abilities

I ruled out Medium and Squarespace as they do not accurately reflect my current abilities as a developer. GitHub Pages is not extendable enough... so why not WordPress?

In terms of my four requirements, WordPress scored low enough on average to be disqualified by this criteria alone but I suppose the main reason I didn't want to build another WordPress blog is because I have been down there. I know that road. I know exactly where it ends. And I know that's not where I want to be.


## The Site

Being a command line junkie I settled on the static markdown blog format, that way I could write, format, and publish each blogpost without having to touch a mouse. Splendid. I went with Jekyll. It's the most popular of the static site generators so in theory there'll be more support.

The next step: plan out a modular development structure for the website.

I decided to split the code into three repositories: blogposts, website and website theme. By isolating the blogposts, I could compensate my lack of a GUI by drafting posts from github.com if I needed to. By seperating the theme from the main site, I could switch to a new theme without having to refactor the entire codebase and I could potentially use either repo for future websites. I chose Hyde by @mdo as a base theme. Forked it, updated it and called it Heidi.

Commiting to this model, the challenge now was to figure out exactly what each repo would look like, how to develop each side-by-side and how to bring them together as the finished site. 

After a little research, I decided to turn the blogpost repo into a submodule of the main site repo. A simple `git submodule update --recursive --remote` would fetch the latest posts. However, this approach proved too clumsy for the theme repo, so instead I developed Heidi as a seperate ruby gem.


## The Server

I had a number of requirements from my server:

1. TLS encryption (https:// instead of http://)
2. A staging domain to make sure the site works in a live environment
3. A reverse-proxy to easy facilitate future capabilities
4. A docker-centric workflow (my latest infatuation)

I began developing the most excessive, scalable, docker server cluster I could imagine. With load balancing, continuous deployment, failover, all spread across as many virtual servers as i desired. By the time I was knee deep in iptables documentation, I realised I'd gone too far. Vowing to return one day to my mega cluster I pushed on, building a simple NGiNX reverse-proxy before finding a superb ready-made solution on GitHub.

The server now consists of five parts:

- The main NGiNX reverse proxy docker container.
- A container that generates and updates the main NGiNX container configuration.
- A container that handles the acquisition and renewal of the TLS encryption certificates.
- A container with my staging website available at https://dev.howlinbash.com
- And finally a container with my live website: https://howlinbash.com


## The Development Environment

To build a seperate theme whilst building a website turned out to be my most disruptive decision. My most puzzling problem to solve.
- Which repo should I commit which code changes to?
- How can I make sure the changes I make to my theme will load on my site?

There were no two ways about it. I needed a bash script.

---

To clarify:

My goal was to build a website, it was not to build a theme. The theme would be a by-product of the website. Ideally as I built the website, the theme would magically build itself behind the scenes, without me having to think about it. This meant that I would need to develop the website from both the theme repo and the website repo almost simultaneously.

So, I built this script.

Below is a description of what each argument does and why I built it.

### Load
> Move posts, pages & config from site repo to theme repo for development.

Loading all my website specific files to the theme directory means that although I'm only commiting code to my theme, to me it looks like I'm working on my actual website from within my website repo.

- I use `sed` to comment out the howlinbash specific settings in my config file.
- I use `git update-index --assume-unchanged` to ignore the howlinbash specific files.

### Serve
> Serve and watch theme or site.

If I serve my theme, the script moves to the theme repo and runs `bundle exec guard` to serve a live-reloading website to localhost.

However, if I serve my site, the script 
- moves to the site repo
- starts a docker container that watches and builds the site to `/web`
- starts a second container that watches and serves the same directory to localhost.

### Test
> Test theme by pushing, pulling and building from local gem server.

To avoid commiting broken code I have to be able to test my theme works when loaded as a gem.

The script 
- loads any changes from the theme config file to the site config file
- starts a docker container with a local geminabox test gem server
- switches all the config variables from the theme name to the test theme name
- makes a test gem and pushes it to the test server
- serves the test site in a docker container to localhost
- reverts all config variables back to the original theme name

### Bump
> Update theme version number, commit changes and post gem to rubygems.org.

When I know that the theme works as a gem and I'm happy with my changes, I bump, commit and publish.

The script
- makes a gem
- bumps the version number in the relevant config files
- pushes the gem to rubygems
- commits the changes to GitHub
- interactively commits the theme config file incase there are any changes to commit
- tags the commit with the version number

### Deploy
> Deploy changes.

Once I'm content with my code, I push it to staging.

The script
- pulls the latest posts from the blogpost repo
- builds the website docker image
- pushes the image to dockerhub
- SSHes into my server
- pulls the latest image from dockerhub
- replaces the staged container with the image
- reloads the server

### To Be Completed...

I have two final functionalities to add to my build script
- add a third option to `serve` to preview live changes to blogposts
- a function to push a completed image from staging to live.

## In Closing

