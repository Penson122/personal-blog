---
title:  "Outages Happen"
---

It took less than a week to cause an outage. Astute readers will notice the mistake from the last post. I'll show it again here:

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
```

Standard copy-paste without thinking, rushing to get stuff working. It's also the pain of not having any peer-review process. What I'd done is not mount the certs directory to the host system. As I'd been experimenting with creating this site and restarting the site as I was pushing updates to the first post and playing with different themes and jekyll configs I had been requesting new certificates the whole time.

I then ran head long into the [let's encrypt rate limiter](https://letsencrypt.org/docs/rate-limits/). So on my final update I was granted with this lovely page:

![Danger ahead!](/assets/images/dangerous-site.png)

Oh no! How peculiar, what on earth had happened to the certs?

```
$ docker logs letsencrypt

...
```

By default `letsencrypt-nginx-proxy-companion` will supply a self-signed cert if it can't create a valid one from lets encrypt. You could see this in the browser:

![self-signed certificate](/assets/images/self-signed.png)

When I looked back at my hacky startup script it was obvious that I was throwing the certs away. What I should have done is:

```bash
docker run --detach \
      --name nginx-proxy \
      --publish 80:80 \
      --publish 443:443 \
      --volume /local/path/certs:/etc/nginx/certs \
      --volume /etc/nginx/vhost.d \
      --volume /usr/share/nginx/html \
      --volume /var/run/docker.sock:/tmp/docker.sock:ro \
      jwilder/nginx-proxy
```

However, I've been throwing all these certs away and now I really need them. I can't make the site http only after lambasting it in my first post!

Docker has to keep these volumes somewhere on the disk, so where? `/var/lib/docker/mounts`:

```
$ ls /var/lib/docker/mounts

uuids
```

Ah. That's not _super_ helpful.

Luckily

![I know unix](/assets/images/its-unix.gif)

The plan:

1. Spin up the containers with the correct mount
1. Find those files in `/var/lib/docker/mounts`
1. Rescue the files and restart the container to get https working again

### Step 1

```
$ ./start.sh

$ ls certs

```

### Step 2

```
$ /find
```

Now that we've found the certs we can verify if they're the correct ones by using `openssl`
```
$ openssl ...
```

Um, that's not it chief. It looks like the default.crt isn't the correct file.

Hmm, well we know that certificate files will have `RSA` as the header, so we could grep for that and see if we find other files:

```
$ grep
```

That's the ticket. `jackpenson.dev.cert` that certainly sounds more like it.

And if we open it up again
```
$ openssl
```

### Step 3

So we'll grab that mount, replace our certs and restart all the things.

```
$ cp -R /var/lib/docker/mounts ...
$ kill.sh && start.sh
```

![jackpenson.dev working](/assets/images/correct-cert.png)

![Huzzah](/assets/images/huzzah.gif)


## Retro

All in all it took longer to deconstruct the outage and turn it into a post than it took to resolve it. Luckily as this had all been on the same host the docker volumes had stuck around. If these were on ephemeral hosts I would have lost these volumes and i'd have had to wait an entire week to get the site back to https. I suppose at that point I really would have had to hang my head in shame and host the site as http for the time being.
