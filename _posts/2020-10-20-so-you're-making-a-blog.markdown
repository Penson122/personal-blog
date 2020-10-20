---
title:  "So You're Making a Blog."
---

# Just a random example

Let's say you're a some-what recently graduated developer and you find yourself tunnel-visioning on particular technologies at work. If this is the case, in order to not go stale, you're going to need to find an outlet where you can experiment with new technologies, and record your findings and opinions.

In order to experiment with new technologies you're going to need to dedicate time and effort to keeping up with the goings on of modern development. This will give you a perfect excuse to feel like you're not wasting your time and energy on reddit and hackernews.

And seeing as you're going to be conducting experiments and learning about cool new things, you're also going to need somewhere to host them so that you can share them on the internet. Because of course, if a stranger online doesn't validate your work, did you do any work at all?

# How I made this blog

## A note to undergraduates

I'm going to be aiming much of this content at people like you. Specifically, I'm going to be making the kind of content I wish I had access to whilst I was in university. I'll be aiming to be making complete guides, with links to repositories with the complete code, which actually build, and stays building (perpetual guarantees notwithstanding).

This blog itself could be hosted completely for free. One of the reasons I've set this blog up in a more complicated way than it strictly needs to be is because I'm testing myself in cloud admin practices and techniques.

If you'd like to host your own blog using Jekyll, or using some other static site builder, go read these docs.

## A brief architectural interlude

This blog is, at the time of writing, hosted at [jackpenson.dev](https://blog.jackpenson.dev).

The tech stack currently looks like:
* Jekyll Static Site
* Containerised with nginx
* Proxied with [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) and [docker-letsencrypt-nginx-proxy-companion](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion)
* Hosted on Digital Ocean

In the very near future I'm going to add:
* Infrastructure managed via Terraform
* Additional static sites (such as a resume)

With the eventual architecture looking something like this:
_images to follow_

## nginx-proxy

Hosting on my own tin (or near enough) gives some distinct advantages over using pre-packaged solutions like GitHub Pages. A key one being this site isn't limited being just a blog. I plan to host multiple subdomains for the various things I hope to make and blog about.

[nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) is a particularly useful tool for the initial phase of this project. It generates reverse proxy
configurations for docker containers running with particular environment variables. The full details on how this works are discussed here.

This means that hosting additional sites becomes very straightforward, simply deploy additional containers and inject the correct environment variables, and hey-presto, that's a new subdomain.

You will of course need to create a new A-Record for that. Which means that for each additional subdomain only two changes are required, adding the container to the list of services (with the right env vars) and updating the DNS config with the new A-Record. Easy peasy.

## Let's Encrypt

There are absolutely no excuses to host a site without https anymore. It's just bad practice. There is a lot of tooling out there which makes this an entirely brain-dead exercise. [docker-letsencrypt-nginx-proxy-companion](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion) is designed to be used in tandem with `nginx-proxy`. Luckily it does JustWork:tm:.

[Refer to the official documentation for how to get nginx-proxy and lets encrypt working together](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion).

## Digital Ocean

I've opted for digital ocean because it offers a great balance between flexibility and foot-gunning. My day job is a cloud admin focusing on aws, kubernetes, chef, hosting various internal tooling and self-hosted SaaS services. I don't need a full Enterprise solution for my personal blog, thank you very much.

## Automate ALL the things!
