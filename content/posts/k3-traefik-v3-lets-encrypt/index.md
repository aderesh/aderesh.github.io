+++
date = '2025-12-11T23:18:00-05:00'
draft = true
title = "Adding Let's Encrypt certificates to Traefik V3 in K3S"
+++

Are you tired of 'Your connection is not private'? Especially if you make changes your workloads ingress and traefik regenerates the certificates and you get the same warning over and over and over. 

Today, we will be looking into how to properly configure https using Let's Encrypt certificates using Traefik V3 running in k3s cluster for your http servers. 

The choosen approach has a few benefits:
- Let's encrypt certificates will be automatically renewed by Traefik
- Certificates will be saved in a volume (Longhorn volume in my case). No losing certificates on ingress recreations
- No need to expose your cluster - we will use [DNS-01](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge) ACME challenge to renew certs

Preqrequisites:
- k3s running Traefik v3
- domain name (e.g. 'deresh.net')

There are two ways I'm aware of to automatically manage certificates in k3s - traefik and cert-manager. Since I already have a Traefik running my Home Assistant stuff with the Let's Encrypt configured and it works really well, I decided to do the same in my k3s cluster. So, Traefik will do all the certificate renewal. 

