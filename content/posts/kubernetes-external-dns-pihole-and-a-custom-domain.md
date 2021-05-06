---
title: "Kubernetes, external-dns, PiHole and a custom domain"
date: 2021-05-06T21:17:56+02:00
draft: true
---

During these days I'm tiding up my homelab and found the neccesity of having an internal domain to expose some apps inside my local network but not to the internet.

For example, I use [Vault](https://www.vaultproject.io/) to store secrets and I want an easy way to access the web-ui rather than using the IP address. The solution in Kubernetes is to create an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) but if I use my domain diomedet.com it will be exposed to the whole internet and I don't want that.

[external-dns](https://github.com/kubernetes-sigs/external-dns) is my tool of choice to handle the synchronization between my Ingresses and the DNS provider, on my local network I use [PiHole](https://pi-hole.net/) to filter all my DNS request and to block some of them.

PiHole already have a "Local DNS Records" section where you could list an arbitrary domain and point it to a specific IP inside or outside your network.
So, if there is a way to make **external-dns** updates this list what I'm trying to achieve it would be possible with a little effort. Unfortunately, at the moment of writing, there is no way to update the list of local DNS records on PiHole programmatically, so we've to find another way to do that.

Messing around with the interface of PiHole, I've noticed that under "Settings -> DNS" you can choose to which DNS server redirect all the incoming requests that hasn't be blocket by the blacklist and besides the classi list of "Upstream DNS Servers" there is also a list of custom upstream DNS servers:

![PiHole DNS Settings](/imgs/posts/kubernetes-external-dns-pihole/pihole-dns.png) 

So, the idea is to create a custom DNS server that can be update by **external-dns** and used by PiHole as an **upstream DNS server**, in this way every ingress with my domain of choice, inside my local network, will be resolved to the IP of my kubernetes cluster.

Great, we've a plan, now it's time to make it real!

<!-- # TODO: talk about the difficulties with CoreDNS and etcd -->
<!-- # TODO: explain that at the moment there is no SSL but it will be added in the future -->