---
title: "Kubernetes, external-dns, PiHole and a custom domain"
date: 2021-05-06T21:17:56+02:00
draft: true
---

During these days I'm tiding up my homelab and found the neccesity of having an internal domain to expose some apps inside my local network but not to the internet.

For example, I use [Vault](https://www.vaultproject.io/) to store secrets and I want an easy way to access the web-ui rather than using the IP address. The solution in Kubernetes is to create an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) but if I use my domain `diomedet.com` it will be exposed to the whole internet and I don't want that.

[external-dns](https://github.com/kubernetes-sigs/external-dns) is my tool of choice to handle the synchronization between my Ingresses and the DNS provider, on my local network I use [PiHole](https://pi-hole.net/) to filter all my DNS request and to block some of them.

PiHole already have a "Local DNS Records" section where you could list an arbitrary domain and point it to a specific IP inside or outside your network.
So, if there is a way to make **external-dns** updates this list what I'm trying to achieve it would be possible with a little effort. Unfortunately, at the moment of writing, there is no way to update the list of local DNS records on PiHole programmatically, so we've to find another way to do that.

Messing around with the interface of PiHole, I've noticed that under "Settings -> DNS" you can choose to which DNS server redirect all the incoming requests that hasn't be blocket by the blacklist and besides the classi list of "Upstream DNS Servers" there is also a list of custom upstream DNS servers:

![PiHole DNS Settings](/imgs/posts/kubernetes-external-dns-pihole/pihole-dns.png) 

So, the idea is to create a custom DNS server that can be update by **external-dns** and used by PiHole as an **upstream DNS server**, in this way every ingress with my domain of choice, inside my local network, will be resolved to the IP of my kubernetes cluster.

Great, we've a plan, now it's time to make it real!

## First things first, we need a DNS server

Scouting between the providers supported by **external-dns** there a bunch of choices that can be self-hosted, something like [PowerDNS](https://www.powerdns.com/) or [CoreDNS](https://coredns.io/), at this point I was like: 
> "mmh interesting, CoreDNS is the one used by kubernetes internally must be a good choice, let's go with it"

A collegue of mine suggested to use **PowerDNS** but I was already set on my path, so I sticked with **CoreDNS**.

To be clear, it's not a bad choice, but might be a little overkill and over engineered for this specific purpose but let's see what were the difficulties that this path include.

In the **external-dns** repo there is a folder `docs/tutorial` with a markdown file for each supported provider (I think each, didn't counted), we're looking for the CoreDNS one: [https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/coredns.md](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/coredns.md)

Is a tutorial for minikube, but ignoring that part, can be used for every kubernetes cluster and bonus point, show us even how to install **CoreDNS**, great two birds with one stone.

If you've opened the file you can see from the very beggining that the birds are not two anymore but three, the more the merrier right, right?!

If you haven't opened the link, let me recap that for you what we need to install:
- CoreDNS (obviously)
- etcd
- another instance of external-dns (you need an instance of external-dns for each dns provider you're going to support)

## Wait, why we need etcd?

This is the way how **external-dns** talks to **CoreDNS**, we have to create a section in the configuration of CoreDNS that tells him to read the value from a certain path on the **etcd** instance we're going to configure and external-dns will update the same path with the information about the Ingresses we're going to create with our internal domain.

Before swtiching to **etcd** directly, **CoreDNS** was using [SkyDNS](https://github.com/skynetservices/skydns) (a service built on top of etcd) to serve these kind of request, so, in the manifest files we're going to see you'll find some refuse of that implementation.

## Install etcd

Let's get down to business and install **etcd**, in the end is a core component of kubernetes there nothing wrong learning more about it. Just to let you know, don't use the internal **etcd** for a user application (like the one we want to install here), is not meant for that.

The tutorial linked above suggest us to use the [etcd-operator](https://github.com/coreos/etcd-operator) and use [https://raw.githubusercontent.com/coreos/etcd-operator/HEAD/example/example-etcd-cluster.yaml](https://raw.githubusercontent.com/coreos/etcd-operator/HEAD/example/example-etcd-cluster.yaml) to create our etcd cluster.
> Great, an operator nothing more simplier than that...

Slow down, the `etcd-operator` repo has been archived more than a year ago, even if for a case like this it could work we don't want to install an operator that is not maintained anymore, so let's see how to manually deploy it.

After searching around I ended up on this documentation page [https://etcd.io/docs/v3.4/op-guide/container/#docker](https://etcd.io/docs/v3.4/op-guide/container/#docker) that shows how to deploy etcd with a single node configuration, prefect is what we need here.

So let's see the final manifest:

*etc-sts.yml*
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
  labels:
    app.kubernetes.io/name: etcd
spec:
  serviceName: etcd
  replicas: 1
  updateStrategy:
    type: OnDelete
  selector:
    matchLabels:
      app.kubernetes.io/name: etcd
  template:
    metadata:
      labels:
        app.kubernetes.io/name: etcd
    spec:
      containers:
        - name: etcd
          image: gcr.io/etcd-development/etcd:latest
          imagePullPolicy: IfNotPresent
          command:
          - /usr/local/bin/etcd
          env:
            - name: ETCD_NAME
              value: node1
            - name: ETCD_DATA_DIR
              value: /etcd-data
            - name: ETCD_LISTEN_PEER_URLS
              value: http://0.0.0.0:2380
            - name: ETCD_LISTEN_CLIENT_URLS
              value: http://0.0.0.0:2379
            - name: ETCD_INITIAL_ADVERTISE_PEER_URLS
              value: http://0.0.0.0:2380
            - name: ETCD_ADVERTISE_CLIENT_URLS
              value: http://0.0.0.0:2379
            - name: ETCD_INITIAL_CLUSTER
              value: "node1=http://0.0.0.0:2380"
          volumeMounts:
            - name: data
              mountPath: /etcd-data
          ports:
            - containerPort: 2379
              name: client
            - containerPort: 2380
              name: peer
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi # we don't need much space to store DNS information
```

We're going to use a `StatefulSet` because `etcd` is a stateful app and needs a volume to persist its data. Rather than the classic `Deploy` with a `StatefulSet` we're certain tha the generated pod will receive always the same name and the volume attached to it will always be the same. [More on the StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

The only noticeable difference between the `etcd` documentation and this manifest files is that we're using environmnet variables instead of configuration flags, I had some trouble getting the flags working and I like more the environment variables, anyway here a list of [etcd configuration flags](https://etcd.io/docs/v3.4/op-guide/configuration/) with the matching variable.

In order to expose `etcd` to the other applications in the cluster we need to create a `Service` too:

*etc-service.yml*
```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd
  labels:
    app.kubernetes.io/name: etcd
spec:
  ports:
    - port: 2379
      targetPort: 2379   
      name: client
    - port: 2380
      targetPort: 2380
      name: peer
  selector:
    app.kubernetes.io/name: etcd
```

Nothing special here, but this complete the manifest needed for our etcd instance.

## Install CoreDNS

Now that we have our **etcd** we can continue with the tutorial and install our custom version of CoreDSN, you can use `helm` to install it or if you want a more deep approach you can use `helm template` to render the file and applying them manually or with kustomize.

Since my homelab is a way to learn more about Kubernetes, I choose to render the file with `helm template` and use [kustomize](https://kustomize.io/) to later apply them.

Whichever way you choose, the important part is to set correctly a couple of option inside the `values.yml` file.

```yaml
# if you don't have RBAC enabled on you cluster I think you can set this to false
rbac:
  create: true

# isClusterService specifies whether chart should be deployed as cluster-service or normal k8s app.
isClusterService: true

servers:
- zones:  
  - zone: .
  port: 53
  plugins:  
  ...
  # all other plugins
  - name: etcd
    parameters: diomedet.internal # insert your domain here
    configBlock: |-
      stubzones
      path /skydns
      endpoint http://etcd:2379
```

The most important part is the last one, we're going to configure the `etcd` plugin and tell **CoreDNS** to look inside the `http://etcd:2379` to find the information about the domain `diomedet.internal` (this is my internal domain)

With these values we can run the command (custom is the name of my release, tha it'll turns out in `custom-coredns`):

`helm template custom coredns/coredns --output-dir . --values values.yaml`

This will create a folder `coredns/template` with 5 files in it:

```bash
coredns/templates
├── clusterrole.yaml
├── clusterrolebinding.yaml
├── configmap.yaml
├── deployment.yaml
└── service.yaml
```

Now the only thing we've to do is to `kubectl apply` these files and we'll end up with a working CoreDNS instance. Working but still not reachable outside the cluster, if you have [MetalLB](https://metallb.universe.tf/) configured you can change the `Service` type from `ClusterIP` to `LoadBalancer` in order to get an IP.
I haven't this feature in my cluster yet, so for now I'm going to use the `NodePort` type, this allows me to use a port of my node and point it to the service.

With `kustomize` there is the concept of patch, so I can create a patch that is going to modify the `service.yaml` file without directly touching it. I prefer this way, so if I have to re-run `helm template ...` I don't have to mind any modification I could possibly have made, because everything is applied with a patch.

`patches/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: custom-coredns
spec:
  type: NodePort
  ports:
  - {port: 53, protocol: UDP, name: udp-53, nodePort: 30053}
  - {port: 53, protocol: TCP, name: tcp-53, nodePort: 30053}
```

Here I tell to use the port `30053` for both `UDP` and `TCP`. With the `NodePort` you can use port from 30000 to 32767 if you do not modify it.

To wrap it up, here my `kustomization.yml` file:

```yaml
kind: Kustomization
apiVersion: kustomize.config.k8s.io/v1beta1
namespace: custom-coredns
resources:
- templates/clusterrole.yaml
- templates/clusterrolebinding.yaml
- templates/configmap.yaml
- templates/deployment.yaml
- templates/service.yaml
- etcd-sts.yml
- etcd-service.yml
patchesStrategicMerge:
- patches/service.yaml
```

If you followed my path you should have all the files to make it work, anyway if you've installed the helm chart directly you can always change the service manifest directly on kubernetes. You can set the `serviceType` in the `values.yaml` file but I didn't find a way to specify the `nodePort` to use so I decided to go with the patch.

## Finally, install external-dns

Now we can finally the instance of **external-dns** that is going to monitor the `Ingress` created with out internal domain.

<!-- # TODO: explain that at the moment there is no SSL but it will be added in the future -->