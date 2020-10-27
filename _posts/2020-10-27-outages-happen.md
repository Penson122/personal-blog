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

Standard copy-paste without thinking, rushing to get stuff working. It's also the pain of not having any peer-review process. What I'd done is not mount the certs directory to the host system. As I'd been experimenting with creating this site and restarting repeatedly as I was pushing updates to the [first post](https://jackpenson.dev/2020/10/20/so-you're-making-a-blog.html) and playing with different themes and jekyll configs I had been requesting new certificates the whole time.

I then ran head long into the [let's encrypt rate limiter](https://letsencrypt.org/docs/rate-limits/). So on my final update I was granted with this lovely page:

![Danger ahead!](/assets/images/dangerous-site.png)

Oh no! How peculiar, what on earth had happened to the certs?

```
$ docker logs letsencrypt

Creating/renewal jackpenson.dev certificates... (jackpenson.dev)
2020-10-27 18:58:37,595:INFO:simp_le:1359: Generating new account key
2020-10-27 18:58:42,414:INFO:simp_le:1387: By using simp_le, you implicitly agree to the CA's terms of service: https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf
2020-10-27 18:58:42,767:INFO:simp_le:1450: Generating new certificate private key
ACME server returned an error: urn:ietf:params:acme:error:rateLimited :: There were too many requests of a given type :: Error creating new order :: too many certificates already issued for exact set of domains: jackpenson.dev: see https://letsencrypt.org/docs/rate-limits/
```

By default `letsencrypt-nginx-proxy-companion` will supply a self-signed cert if it can't create a valid one from lets encrypt. You could see this in the browser:

![self-signed certificate](/assets/images/self-signed.png)

When I looked back at my hacky startup script it was obvious that I was throwing the certs away. What I should have done is:

```bash
docker run --detach \
      --name nginx-proxy \
      ...
      --volume /local/path/certs:/etc/nginx/certs \
      ...
      jwilder/nginx-proxy
```

However, I've been throwing all these certs away and now I really need them. I can't make the site http only after lambasting it in my first post!

Docker has to keep these volumes somewhere on the disk, so where? `/var/lib/docker/volumes`:

```
$ ls /var/lib/docker/volumes/
03827a387f60ea52473ad59f9f95bae717631746dfd9567b5115d26f606110df
039d87e378a2f25213526a97a7f7cf52484e99765c4849b8c90c33888657c697
0501865dd599f14e59a8858c8e001dc976b1c453d7149d9368373a8623625b2b
05ba1e5becc3753ee22669032e9548ff1553895aa44b456b496e90e12c58433e
0631b0649b75ecccb9984011b2ec7add79c03a08ab515dce1c103252ca669bb6
0845745ca551d5c52987d6bd0c23872d75fd92ee0b5ca0f27449dd7a58565244

...
```

Ah. That's not _super_ helpful.

That's truncated as there are apparently 105 orphaned mounts:
```
$ find /var/lib/docker/volumes/* -maxdepth 0 -type d | wc -l
105
```

So, it's probably not a great idea to manually walk through this sea of UUIDs trying to find a valid cert.

Luckily

![I know unix](/assets/images/its-unix.gif)

The plan:

1. Spin up the containers with the correct mount
1. Find those files in `/var/lib/docker/volumes`
1. Rescue the files and restart the container to get https working again

### Step 1

```bash
$ ./start.sh
4063e04d164c62c5f0c7dee968120cd9f3f32bdebb510280d81f65e43ae6e4fe
a9e6a7cde043755e8186261a1b5cbdf0aa376a6deac5146537b504aece4fc9bc
0b423a9e141a40d504cf4059f20d18164ea25e52610a79a29c7b357866e91360

$ ls /local/path/certs

accounts  jackpenson.dev  default.crt  default.key  dhparam.pem
```

### Step 2

```
$ find /var/lib/docker/volumes -name default.crt

/var/lib/docker/volumes/03827a387f60ea52473ad59f9f95bae717631746dfd9567b5115d26f606110df/_data/default.crt
/var/lib/docker/volumes/7730da5d87bcb2e7dc9d6dc1b5f8cc0d6a1eb4065c7d7423ae5712b191524111/_data/default.crt
/var/lib/docker/volumes/ac4fd106d0c2466fb30b8e4126818126c168b4161705d03fb4f3b3803cb83f1a/_data/default.crt
/var/lib/docker/volumes/8ca009d92fccff3046137b77c1b5bccd6d9f4ff7b2870134199fe1dda3742a48/_data/default.crt
```

Now that we've found the certs we can verify if they're the correct ones by using `openssl`
```
$ openssl x509 -in default.crt -noout -issuer
issuer=CN = letsencrypt-nginx-proxy-companion
```

Um, that's not it chief. It looks like the default.crt isn't the correct file.

What do these files look like?

```
$ cat default.crt

-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

Hmm, well we know that certificate files will have `CERTIFICATE` as the header, so we could grep for that and see if we find other files:

```
$ grep -rnw -e 'CERTIFICATE' /var/lib/docker/volumes

/var/lib/docker/volumes/6ef9ac7cde6b4cac3d211cd9a8509ea0c5b550eae90743d400f6480e92c18a50/_data/default.crt:30:-----END CERTIFICATE-----
/var/lib/docker/volumes/c4c50b3dd0779e13b1a9abe254ef280a58740c8698d72cc8a56e4e23d4ca4d69/_data/jackpenson.dev/chain.pem:1:-----BEGIN CERTIFICATE-----
```

That's the ticket. `jackpenson.dev/chain.pem` that certainly sounds more like it.

Oh wait, we've got a `jackpenson.dev` directory in `/local/path/certs`. I wonder what's in there:

```
$ ls /local/path/certs/jackpenson.dev
account_key.json  account_reg.json
```

Ahah. It looks like the `chain.pem` is part of the _real_ certificate setup.

If we go have a look in that `_data` directory we find a `jackpenson.dev.crt` too, excellent news!

And if we open that one up:
```
$ openssl x509 -in jackpenson.dev.crt -noout -issuer
issuer=C = US, O = Let's Encrypt, CN = Let's Encrypt Authority X3
```

### Step 3

So we'll grab that mount, replace our certs and restart all the things.

```
$ cp -R /var/lib/docker/volumes/c4c50b3dd0779e13b1a9abe254ef280a58740c8698d72cc8a56e4e23d4ca4d69/_data /local/path/certs
$ kill.sh && start.sh
```

![jackpenson.dev working](/assets/images/correct-certs.png)

![Huzzah](/assets/images/huzzah.gif)


## Post-mortem

All in all it took longer to deconstruct the outage and turn it into a post than it took to resolve it. Luckily as this had all been on the same host the docker volumes had stuck around. If these were on ephemeral hosts I would have lost these volumes and I'd have had to wait an entire week to get the site back to https. I suppose at that point I really would have had to hang my head in shame and host the site as http for the time being.

Luckily, in my laziness I hadn't torn down the host yet. I will need to do so soon when I write terraform for the digital ocean environment. Hopefully this will remind me to take backups before I delete it all.
