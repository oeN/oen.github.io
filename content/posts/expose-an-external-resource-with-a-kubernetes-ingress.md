---
title: "Expose an external resource with a Kubernetes Ingress"
date: 2021-05-02T21:50:31+02:00
draft: false
---

The other night I was wandering around my homelab and noticed that my new home assistant instance wasn't properly exposed to the internet. Some months ago home assistant (from now on HA) found its new permanent home, a Rasperry PI 4 4GB; since we're in lockdown I didn't need to reach HA outside my local network.

I'm still working from home (and love it) but it's time to fix this now. While choosing which method was the best to expose it, I had an idea, why not use an Ingress in my Kubernetes cluster?

It could seem a little odd, but hear me out:

- I already have a Kubernetes cluster, with [cert-manager](https://cert-manager.io/docs/) configured and working
- I'm planning to move some service into the cluster and they'll need to reach HA anyway
- I want to learn as much as I can about Kubernetes and this seemed a good chance

So, in order to achieve my goal, I basically need one thing how I can point a Kubernetes Service to an external resource?

Wait, why a Service? Didn't you mention an Ingress before? Yes, you're right but bear with me.

In Kubernetes, an Ingress needs a service where to redirect all the requests it receives, so in order to properly expose HA with an Ingress, we need to create a Service that can point to it.

Usually, a Service is used to expose a set of Pods that runs an application, in this way if other applications need to talk to each other they don't need to use their pod names (that could change in a various way) but they can simply use the Service name.

When we create a Service another resource is automatically created, an Endpoint, and this endpoint includes the references (all the IP addresses) of the Pods that match the selector we've specified in the Service selector spec.

The Endpoint created in this way will have the same name as the Service we've created.

But here's the thing, we can create a Service without a selector and if we do that we also need to create an Endpoint ourselves and this endpoint could point to an IP outside the Kubernetes cluster.

So let's see the YAML we need to create in order to properly expose HA using a Kubernetes Ingress.

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

Note: we set the clusterIP property to None on purpose, this tells Kubernetes to not provide an IP to this Service, we don't need it. A Service like this is know also as headless service.

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

Now if we run `kubectl apply -f ...` for all these three file after cert-manager has finished its work, we'll end with a domain with a valid certificate that points to our Home Assistant instance outside of Kubernetes.

And remember that I told you that I want to move some service that uses HA inside my k8s cluster? Well, now all I have to do is deploy them and use home-assistant as the URL for HA instead of using the IP address.

For sake of completion, if you already have a local domain for your HA instance you can skip the creation of the Endpoint and use directly the [Service's `externalName` property](https://kubernetes.io/docs/concepts/services-networking/service/#externalname). 