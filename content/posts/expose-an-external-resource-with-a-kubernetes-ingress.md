---
title: "Expose an external resource with a Kubernetes Ingress"
date: 2021-05-02T21:50:31+02:00
draft: false
tags:
- kubernetes
- k8s
- homelab
- home-assistant 
---

The other night I was wandering around my homelab and noticed that I didn't correctly expose my new home assistant instance to the internet. Some months ago, home assistant (from now on HA) found its new permanent home, a Rasperry PI 4 4GB; since we're in lockdown, I didn't need to reach HA outside my local network.

I'm still working from home (and love it), but it's time to fix this now. While choosing which method was the best to expose it, I had an idea, why not use an Ingress in my Kubernetes cluster?

It could seem a little odd, but hear me out:

- I already have a Kubernetes cluster, with [cert-manager](https://cert-manager.io/docs/) configured and working
- I'm planning to move some service into the cluster, and they'll need to reach HA anyway
- I want to learn as much as I can about Kubernetes, and this seemed a good chance

So, to achieve my goal, I need one thing: how can I point a Kubernetes Service to an external resource?

Wait, why a Service? Didn't you mention an Ingress before? Yes, you're right but bear with me.

In Kubernetes, an Ingress needs a Service to redirect all the requests it receives, so to properly expose HA with an Ingress, we need to create a Service that can point to it.

Usually, a Service is used to expose a set of Pods; in this way, if different applications need to talk to each other, they don't need to use their pod names (that could change variously), but they can use the Service name.

When we create a Service, another resource is automatically created, an Endpoint. This Endpoint includes the references (all the IP addresses) of the Pods that match the selector we've specified in the Service selector spec.

The Endpoint created in this way will have the same name as the Service we've created.

But here's the thing, we can create a Service without a selector, and if we do that, we also need to create an Endpoint ourselves, and this Endpoint could point to an IP outside the Kubernetes cluster.

So let's see the YAML we need to create to expose HA using a Kubernetes Ingress.

_endpoint.yml_

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: home-assistant
subsets:
- addresses:
  - ip: 1.1.1.1 # Insert your home-assistant IP here
  ports:
  - name: ha
    port: 8123
    protocol: TCP
```

_service.yml_

```yaml
apiVersion: v1
kind: Service
metadata:
  name: home-assistant
spec:
  ports:
  - name: ha
    port: 80
    protocol: TCP
    targetPort: 8123
  type: ClusterIP
  clusterIP: None
```

Note: we set the `clusterIP` property to `None` on purpose; this tells Kubernetes not to provide an IP to this Service; we don't need it. A Service like this is also known as headless Service.

_ingress.yml_

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  name: home-assistant
spec:
  rules:
  - host: ha.awesome-domain.com # insert your domain here
    http:
      paths:
      - backend:
          serviceName: home-assistant
          servicePort: ha
        path: /
  tls:
  - hosts:
    - ha.awesome-domain.com # insert your domain here
    secretName: home-assistant-tls
```

Now, if we run `kubectl apply -f ...` for all these three files after cert-manager has finished its work, we'll end with a domain with a valid certificate that points to our Home Assistant instance outside of Kubernetes.

And remember that I told you that I want to move some service that uses HA inside my k8s cluster? Now, all I have to do is deploy them and use `home-assistant as the URL for HA instead of using the IP address.

For the sake of completion, if you already have a local domain for your HA instance, you can skip the creation of the Endpoint and use the [Service's `externalName` property](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) directly. 