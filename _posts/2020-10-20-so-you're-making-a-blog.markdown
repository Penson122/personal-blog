---
title:  "So You're Making a Blog."
---

Let's say you're a some-what recently graduated developer and you find yourself tunnel-visioning on particular technologies at work. If this is the case, in order to not go stale, you're going to need to find an outlet where you can experiment with new technologies, and record your findings and opinions.

In order to experiment with new technologies you're going to need to dedicate time and effort to keeping up with the goings on of modern development. This will give you a perfect excuse to feel like you're not wasting your time and energy on /r/programming and hackernews.

And seeing as you're going to be conducting experiments and learning about cool new things, you're also going to need somewhere to host them so that you can share them on the internet. Because of course, if a stranger online doesn't validate your work, did you do anything at all?

# How I made this blog

## A note to undergraduates

I'm going to be aiming much of this content at people like you. Specifically, I'm going to be making the kind of content I wish I had access to whilst I was in university. I'll be aiming to be making complete guides, with links to repositories with the complete code, which actually build, and stays building (perpetual guarantees notwithstanding).

This blog itself could be hosted completely for free. One of the reasons I've set this blog up in a more complicated way than it strictly needs to is because I'm testing myself in cloud admin practices and techniques.

If you'd like to host your own blog using Jekyll, or using some other static site builder, [go read these docs](https://docs.github.com/en/free-pro-team@latest/github/working-with-github-pages).

## A brief architectural interlude

This blog is, at the time of writing, hosted at [jackpenson.dev](https://jackpenson.dev).

The tech stack currently looks like:
* Jekyll Static Site
* Containerised with nginx
* Proxied with [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) and [docker-letsencrypt-nginx-proxy-companion](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion)
* Hosted on Digital Ocean

In the very near future I'm going to add:
* Infrastructure managed via Terraform
* Additional static sites (such as a resume)

Diagrammatically:

![Architecture Diagram](/assets/images/architecture-diagram.jpg)

## nginx-proxy

Hosting on my own tin (or near enough) gives some distinct advantages over using pre-packaged solutions like GitHub Pages. A key one being that this site isn't limited hosting just a blog. I plan to host multiple subdomains for the various things I hope to make and blog about.

[nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) is a particularly useful tool for the initial phase of this project. It generates reverse proxy
configurations for docker containers running with particular environment variables.

This means that hosting additional sites becomes very straightforward, simply deploy additional containers and inject the correct environment variables, and hey-presto, that's a new subdomain.

You will of course need to create a new A-Record for that. Which means that for each additional subdomain only two changes are required, adding the container to the list of services (with the right env vars) and updating the DNS config with the new A-Record. Easy peasy.

## Let's Encrypt

There are absolutely no excuses to host a site without https anymore. It's just bad practice. There is a lot of tooling out there which makes this an entirely brain-dead exercise. [docker-letsencrypt-nginx-proxy-companion](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion) is designed to be used in tandem with `nginx-proxy`. Luckily it does JustWork:tm:.

[Refer to the official documentation for how to get nginx-proxy and lets encrypt working together](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion).

## Digital Ocean

I've opted for digital ocean because it offers a great balance between flexibility and foot-gunning. My day job is a cloud admin focusing on aws, kubernetes, and chef, hosting various internal tooling and self-hosted SaaS services. I don't need a full Enterprise solution for my personal blog, thank you very much.

However the current digital ocean environment is absolutely a snowflak. The firewall, dns, and droplet are all artisanal hand-made works of art. This will all be redone in terraform so that it can be automatically applied and updated.

## Automate ALL the things!

If you haven't noticed yet, I am a big fan of automation. It allows me to feel like I'm being productive and working on things without actually making anything. In all seriousness however there are some key pieces of automation which I feel is worth talking about.

![Automate ALL the things](/assets/images/automate-all-the-things.png)

### Continuous Integration

I'm sure I don't need to convince anyone why this is necessary, but just in case, [start here](https://martinfowler.com/articles/continuousIntegration.html). This blog is hosted on [github](https://www.github.com/penson122/personal-blog) and new blog entries will be added via PRs.

I'm currently using GitHub Actions as my CI task runner largely because I've never used it before. It seems like a very lightweight solution, doesn't require additional accounts, and JustWorks:tm: for my needs. I do plan on doing some kind of comparative analysis of the CI platforms with free tiers, some are certainly better than others at certain things.

### Continuous Deployment

At this current moment the website is not being continuously deployed. This saddens me greatly but is definitely going to be the next thing I work on.

As this is just a bunch of docker containers, the plan is to create a new repository for all terraform with a docker-compose file for deploying them. I reckon it can be as simple as having a github action on any of the subdomain sites which uses `ssh -t` to restart the images with the latest tags on the droplet.

What I'm doing right now is using two hacky scripts:

```bash
docker run --detach \
      --name nginx-proxy \
      --publish 80:80 \
      --publish 443:443 \
      --volume /etc/nginx/certs \
      --volume /etc/nginx/vhost.d \
      --volume /usr/share/nginx/html \
      --volume /var/run/docker.sock:/tmp/docker.sock:ro \
      jwilder/nginx-proxy

docker run --detach \
      --name letsencrypt \
      --volumes-from nginx-proxy \
      --volume /var/run/docker.sock:/var/run/docker.sock:ro \
      --env "DEFAULT_EMAIL=encrypt@jackpenson.dev" \
      jrcs/letsencrypt-nginx-proxy-companion

docker run --detach \
      --name blog \
      --env "VIRTUAL_HOST=jackpenson.dev" \
      --env "VIRTUAL_PORT=8080" \
      --env "LETSENCRYPT_HOST=jackpenson.dev" \
      ghcr.io/penson122/personal-blog
```

```bash
docker kill $(docker ps -a -q)
docker rm $(docker ps -a -q)
```

And on the box:
`$ ./kill.sh && ./start.sh`

So what I'll end up with is a docker-compose definition of all the containers and their config and a restart script. Then all I'll need to do is

`$ ssh -t me@digital-ocean "restart.sh"`

Of course this doesn't guarantee zero-downtime releases and would not be suited to high availability environments. But it doesn't need to be. If I ever get to the point where high availability is a concern I doubt I'll be running all the subdomains from a single $5 droplet.

## Endings are weird

I plan on writing about CI/CD practices, DevOps, maybe even something around art and programming. Particularly I'd like to play around with parsing or constructing conceptual art pieces like those of [Laurence Weiner](https://www.guggenheim.org/artwork/artist/lawrence-weiner). Maybe even some generative art?

So yeah thanks for reading the first of I hope, at the very least, several posts.
