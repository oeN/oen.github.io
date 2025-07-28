---
title: "Forecastle: my services Dashboard of choice"
# to keep the old link
permalink: /posts/forecastle-my-services-dashboard-of-choice/
date: 2021-05-12T21:59:20+02:00
draft: false
tags:
- kubernetes
- k8s
- homelab
---

Some weeks ago, I started the migration of my homelab from many docker-compose files to a more organized and reliable Kubernetes cluster.

In both these scenarios, I needed a way to collect all the links to the various services I have in one place. In the docker-compose implementation, I've used [Heimdall](https://github.com/linuxserver/Heimdall), a great tool that gets the job done, but it requires some manual work. With the Kubernetes version of my homelab, I'm trying to automate as many tasks as possible.

I don't remember what the search phrase used to find this was (I couldn't replicate the results), but I come across [Forecastle](https://github.com/stakater/Forecastle) and decide to give it a try.

The installation is pretty simple, follow the steps described on the repository, and you're good to go. I went for the "Vanilla Manifests" path and ended copy the manifest in my homelab repo to inspect it better.

Looking into the manifest, I see that it will create the classic resources, Deployment, Service, ConfigMap, etc. And also a CRD called **ForecastleApp** and later on, we'll see how it can be used.

As mentioned in my previous post, where I described [my journey with **external-dns** and **Pi-hole**](./kubernetes-external-dns-pihole-and-a-custom-domain.md), I use [Kustomize](https://kustomize.io/) to manage all my manifest. After downloading the **Forecastle** manifest, I've added it to a `kustomization.yml` file:

```yaml
kind: Kustomization
apiVersion: kustomize.config.k8s.io/v1beta1
namespace: Forecastle
resources:
- forecastle.yaml
- ingress.yml
patchesStrategicMerge:
- configmap-patch.yml
```

Other than the `forecastle.yaml` file (this is the one copied from the repo), you can notice the `ingress.yml` and the `configmap-patch.yml`

The ingress is nothing special, just an internal domain that points to the `forecastle` service, created with the base manifest:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: forecastle
spec:
  rules:
  - host: hub.diomedet.internal
    http:
      paths:
      - backend:
          serviceName: forecastle
          servicePort: http
        path: /
```
Note: if you want to know how to create an internal domain, check the post I've linked above ;)

And then, we have the `configmap-patch.yml` containing a patch applied to the ** Forecastle** config map. Using this patch method allows me not to worry about the base file, which I can re-download every time I want. Kustomize will apply the patch until the config map name does not change.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: forecastle
data:
  config.yaml: |-
    crdEnabled: true
    namespaceSelector:
      any: true
```

This patch is essential. It tells **Forecastle** to watch all the namespaces and to enable the CRDs.

For me, the `crdEnable: true` option is essential because I have some services outside the cluster that I want to have in my dashboard; this allows me to create a **ForecastleApp** like this one:

_unraind.yml_
```yaml
apiVersion: forecastle.stakater.com/v1alpha1
kind: ForecastleApp
metadata:
  name: unraid
spec:
  name: Unraid
  group: Infrastructure
  icon: <icon_url>
  url: http://<unraid_url>:3080
  networkRestricted: true
```

For all the services inside the cluster that needs to be in the main dashboard, I use the annotations on the Ingress, like:

_home-assistant/ingress.yml_
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    forecastle.stakater.com/expose: true
    forecastle.stakater.com/appName: Home Assistant
    forecastle.stakater.com/group: Home Automation
    forecastle.stakater.com/icon: https://<ha_url>/static/icons/favicon-apple-180x180.png
  name: home-assistant
spec:
  rules:
  - host: <ha_url>
    http:
      paths:
      - backend:
          serviceName: home-assistant
          servicePort: ha
        path: /
  tls:
  - hosts:
    - <ha_url>
    secretName: home-assistant-tls
```

Technically, Home Assistant is not inside the cluster yet, but this Ingress fits the example I need. If you want to know how I've exposed Home Assistant through the cluster, check my other post on [how to expose an external resource with Kubernetes](./expose-an-external-resource-with-a-kubernetes-ingress.md).

In conclusion, now I have two ways of automatically create a new entry in my service dashboard, no more manual insert!

Heimdall does something more than collects links; for some apps, likes Pi-hole, it can show you some stats (like the number of queries blocked) right in the dashboard without even open the link. Even if I had many apps supported by Heimdall, I never used that feature so much, so I'm not going to miss it.

There is one thing I'm going to miss about Heimdall; it comes with many icons you only have to pick.

Anyway, I'm not saying that **Forecastle** is better than **Heimdall** it only suits my use case better.

See you the next time, bye!
