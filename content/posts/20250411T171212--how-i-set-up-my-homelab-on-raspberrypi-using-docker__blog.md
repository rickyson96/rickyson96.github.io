+++
title = "How I set up my homelab on RaspberryPi using docker"
date = 2025-04-11T17:12:00+07:00
tags = ["blog"]
draft = false
+++

## Introduction {#introduction}

I've been owning Raspberry Pi 4 for a while now and had been
successfully setting up simple homelab services for my own usage. This
include deploying [pihole](https://pi-hole.net/) + [dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy) for my adblocker and
installing [nextcloudpi](https://nextcloudpi.com/) alongside them.

For setting up adblocking + avoiding dns poisoning, I followed one of
[my colleague's write-up](https://segmentationfault.xyz/posts/2022-07-30-pihole-dnscrypt.html) (huge shoutout to Mas Tonny!). And for the
nextcould part, I used the nextcloudpi setup wizard.

To bring it on the go, I've set up [tailscale](https:tailscale.com) for VPN purposes and
enable my devices to connect with each other.


## Problem {#problem}

One thing that I should be paying attention to, which I haven't, is
backup. This holds true, especially for my raspberry pi setup. For
context, I run my old raspberrypi with single sd-card without any
backup whatsoever. The result? I've corrupted my storage twice, not
something I'm proud of.

Luckily, I haven't gone cold-turkey on backing up all of my data using
nextcloud. Most of it still lives in Google Drive. And I'll need to
have backup solution in place before going full in nextcloud
(especially since my gdrive storage is reaching it's limit).

That being said, one thing that's really sucks is that I need to redo
all of those setup again, everytime the sd-card corrupts. Hence the
need of automated solutions.


## Alternatives {#alternatives}

We have multiple choices of automated installation system that I know of:

-   Ansible (a.k.a [ThePrimeagen choice](https://frontendmasters.com/courses/developer-productivity-v2/))
-   Nix (or Guix)
-   Docker (or other container based installations)

For me, I've opted to use Docker, mainly because that's what I'm most
familiar with. I don't have any prior knowledge of Ansible. I've used
Nix and NixOS in the past, but I'm not really well versed in it, and
I've written a bunch of Dockerfiles in the past.

Understanding docker more in depth also benefits me since I'm a
backend web developer and most of the system I've been working on in
the server is containerized.

Lastly, tailscale is really generous on providing the [guide to getting
started with tailscale and docker](https://tailscale.com/kb/1282/docker). It's a good writeup and video,
which I encourage you to go check it out!


## The System {#the-system}

The system currently consists of [pihole](https://pi-hole.net/), [stirling-pdf](https://www.stirlingpdf.com/), and [syncthing](https:syncthing.net).
All the basic setups follows [tailscale's guide](https://tailscale.com/kb/1282/docker), which I won't repeat
here since they do a better job at that. I'll write about what I do
differently here, since we're dealing with DNS resolution inside
docker (with pihole).

Usually, this containerized system mount 2 different container's
network into 1, as if running one of them as sidecar. This is achieved
using the `network_mode: service:tailscale-sidecar`. This way, we treat
those 2 containers as if they're on the same network. The main
container then can initialize connection to our devices using
tailscale.

For my pihole container, I decided to mount 3 containers into 1.
`pihole-ts` for tailscale, `pihole` for pihole, and `dnscrypt` for
dnscrypt-proxy. I decided it that way so that `pihole` can talk directly
with `dnscrypt` for resolution purposes without needing to go outside to
tailscale for that.

I've also added port expose on the `pihole` group, so that my raspberry
pi can also benefit from those DNS resolver. We do it in the
`pihole-ts` container, since the other container attaches themselves to
the tailscale container, and exposing the parent also exposes the
others.

The `pihole-ts` also uses custom script to boot `tailscaled`. The idea is
to generate custom `certs.pem` file so that we can use `https` for our
pihole connections. This scripts got it's idea from
[ericwastaken/docker-tailscale-sharing-certs](https://github.com/ericwastaken/docker-tailscale-sharing-certs), though you might be
better off by directly using his container. I might do that later if I
encounter cert expiring issues, since it also take care of the
automatic certs renewal, etc. His container doesn't have pihole
specific requirements though, which is combining the `.crt` and `.key`
into single file `.pem`.

I've also utilized docker-compose environment resolution. This way, I
can encrypt the environments (which contains sensitive data) and share
the rest of the configurations. This also works outside of the
`environment` key, which is displayed by using the `${PWD}` environment on
`volumes`. I think this is very neat if we need other parts to be
configured by environments (such as custom exposed ports for the
compose service, etc).

The dns parts is also the courtesy of running pihole inside docker
container. The default docker's dns resolver is unable to find the
correct server without us manually configure it. Luckily, we're
running our apps inside tailscale, thus we only need `100.100.100.100`
for our tailscale services, and pihole machine's tailscale IP which
can be obtained in the main machines page and look for our `pihole`
machine.

That's it! If you haven't check out the [tailscale on docker's guide](https://tailscale.com/kb/1282/docker),
bunch of the stuff that I talked about might make less sense. I
encourage you to go check it out (it's the third time I mentioned
this). And while we're on this, you should go check out their [deep
dive](https://www.youtube.com/watch?v=tqvvZhGrciQ) too. It's a video worth watching, simply because you'll get
deeper understanding on docker's network (unless you're a pro
already).

My full configurations can be found [here](https://github.com/rickyson96/raspberrypi-services/blob/9676f9222f4a47a9fab8fab6bf22d284192d8d79/docker-compose.yaml), in case you're wondering
about my setup.
