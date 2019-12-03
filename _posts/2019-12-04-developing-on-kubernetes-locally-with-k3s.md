---
layout: post
title: "Developing on Kubernetes locally with k3s"
author: Staffan Olsson
date: 2019-09-23
categories:
---

Once you get used to `kubectl` tab completions of pod names that are almost as fast as local paths,
you'll want to run kubernetes on your laptop rathern than develop on a managed remote cluster.

This talk will demo a basic "inner development loop" using Docker builds and a local image registry running on k3s.

## What I've been looking for

1. A sample app, "inner loop" example
   -
2. `kubectl logs myapp-` _press tab_
3. Minimal fan noise

Critera 3 is not only the reason to look for a lightweight Kubernetes setup,
it's also the reason I'm running Mac. I'd prefer to run Ubuntu but I don't have time to find the right hardware.

## Our evaluation criterias

1. Compatibility with our production workloads
2. Compatibility with our kubectl workflows (top, logs)
3. Lighness (non-scientifically)
   - Time to provision
   - How well we felt that our hardware resources converted to pod resources
     - (containerd vs. dockerd)
4. Support for a local image registry
   - Which to us means an in-cluster, ephemeral docker registry

## Why we chose k3s

We evaluated [Minikube](https://minikube.sigs.k8s.io/),
[microk8s](https://microk8s.io/) and [k3s](https://k3s.io/), in that order.
We also briefly tried [kind](), but found it too geared towards CI.

### minikube

Despite the recent emoji explosion, we had a feeling we need to look further.
If you switch Minikube to running [containerd]() from running Docker,
it seems to reduce resource usage,
but it also becomes clear that the Minikube community struggles with quite a bit of legacy.
(Right now I can't remember what feature it was that didn't work with Containerd at the time we tested)

### containerd vs. dockerd


### microk3s

With microk8s we found odd issues with stacks that have long been robust on GKE,
for example our very basic [MariaDB](https://github.com/Yolean/kubernetes-mysql-cluster) cluster.
In the face of a few such incompatibilities we didn't really troubleshoot, we just looked around again.

### k3s

```bash
# more about multipass later
multipass launch -n k3s -c 2
# then
multipass shell k3s
# ... continue inside the VM
sudo -s
# and run the installer from https://k3s.io/
journalctl -ef -u k3s
k3s kubectl get pods --all-namespaces

# back on the host again ...

# We need a DNS name because the machine IP will change per launch
VM_IP=$(multipass info k3s | grep 'IPv4' | awk -F' ' '{print $2}')
echo "$VM_IP  k3s.local" | sudo tee -a /etc/hosts

export KUBECONFIG=$(pwd)/k3s-demo-kubeconfig.yaml

multipass exec k3s -- sudo cat /etc/rancher/k3s/k3s.yaml \
  | sed "s|127.0.0.1|k3s.local|" \
  > $KUBECONFIG
kubectl get pods --all-namespaces
```

Our first impression of k3s was that [install.sh](https://github.com/rancher/k3s/blob/master/install.sh) wasn't scary and that
all workloads we have in production today just worked as-is.
Minikube never disappointed in that regard, but my experience with Minikube is that you've always needed to tweak things
(for example [Knative's](https://knative.dev/docs/install/knative-with-minikube/#creating-a-kubernetes-cluster) setup is like a "flavor" of Minkube).
With the risk of coming across as a fanboy I have to say that k3s induced a kind of never-look-back feeling.

## What is k3s

K3s doesn't come with provisioning for your OS.

What k3s targets that we don't actually care about is to be persistent.
Server and nodes survive machine restarts and can be upgraded.

The [k3s docs](https://k3s.io/) has a nice graphic.

## How to select a VM on Mac

Here I hope that the audience has more ideas than we've come up with.

From our experiences with Minikube we've actually ruled out VirtualBox.
Switching to [Minikube's hyperkit driver](https://minikube.sigs.k8s.io/docs/reference/drivers/hyperkit/) reduces CPU load significantly.
Thus for k3s we found ourselves looking for linux running on Hyperkit,
and the (only?) option we found was [Multipass](https://github.com/CanonicalLtd/multipass/).

### Our experiences with Multipass

The only issue we've had is a few occasions where the VM had clock skew after the host had been in sleep.
We haven't seen this happening since all developers upgraded to Docker for Mac 19.03+.
Also the CLI has an annoying gotcha where many commands when invoked without a machine name immediately starts to provision a "default" machine.

## What about k3os, k3d or vanilla docker images

We have [an issue](https://github.com/y-stack/ystack/pull/17) right now that triggered some temptation to look into dedicated k3s hosting,
namely [k3os](https://github.com/rancher/k3os/) and [k3d](https://github.com/rancher/k3d/).

As for k3os we're yet to find a simple way to run it on Mac.

As for k3d, do we really want to add a layer of containerization around k3s? Also ...

### k3s already runs in docker

There's a [docker-compose.test.yml](https://github.com/y-stack/ystack/blob/master/docker-compose.test.yml) in the ystack repo that brings up a fully functional k3s
using the project's official docker image.
A flaw we found, and never got to troubleshooting, is that `kubectl logs` doesn't work.
There are two reasons we're sticking with Multipass for the time being:

 - When you delete a VM it's really deleted. Containers, networks, volumes etc. in Docker are not that simple to wipe out.
 - We found CPU load to be higher than with multipass for the same provisioning steps (which might however have been because it was faster).

Given how well k3s performs in docker we're quite tempted to use it for CI, as ephemeral cluster.

## What works out of the box

 - Automatic volume provisioning (with a default storage class)
 - Ingress
   - With the Multipass VM, got to be machine-dependent
 - `kubectl top`
 - `kubectl logs` (not with the docker-compose setup)

So finally ....

# k3s demo time

I'll try to run a subset of our [development stack]() consisting of:

 * k3s
 * a docker registry running in k3s

And, locally on my machine, `kubectl` + Docker + [Skaffold]().

## First of all, the demo app


```bash
for f in helloworld.go go.mod Dockerfile; do curl -fL -o sample-app/$f --create-dirs https://github.com/knative/docs/raw/master/docs/serving/samples/hello-world/helloworld-go/$f; done

docker build -t helloworld sample-app
```

Ok so now we can do `docker run`, but what's the name of this meetup again?

## Provisioning

Run a linux VM.

Run k3s install.

If you're ok with someone else's automation for this, feel free to look at our [provision](https://github.com/y-stack/ystack/blob/master/bin/y-cluster-provision-k3s-multipass) script.

## Adding a docker registry

Skaffold and other Google-guided dev tooling out there is heavily biased towards a central image repository.
That's probably because they are biased towards development in a managed cluster.
With a local cluster it appears quite insane to first push your images to a remote registry and then pull them back to your local VM.

There seems to be a small but persistent fraction of communities like Minikube, Knative and k3s that stubbornly files issues arguing for "private" registry support.

K3s recently introduced [custom Containerd registry config](https://rancher.com/docs/k3s/latest/en/installation/airgap/#create-registry-yaml) which is really neat if you want to set up "mirrors", i.e. re-map hostnames and select http or https.

Let's borrow Yolean's registry setup but use ephemeral storage:

```
kubectl apply -k github.com/y-stack/ystack/registry/generic?ref=e3d0a7e
kubectl patch deployment registry --patch '{"spec":{"replicas":1,"template":{"spec":{"containers":[{"name": "docker-v2","env":[{"name":"REGISTRY_STORAGE","value":"filesystem"}]}]}}}}'
kubectl get pods
```

### What about using the cluster's docker daemon

Minikube's support for DOCKER_HOST works really well,
but the problem is that Kubernetes behaves differently with pre-pulled images.
You actually have to fiddle with `imagePullPolicy` in most of your yaml, wich goes agains "production-like".

### How do we push builds to the registry?

Note that Docker enforces https for all non-localhost addresses,
so you need to add an [insecure registry](https://docs.docker.com/registry/insecure/). In Docker for Mac you can edit `docker.json` from Preferences > Docker Engine.

Let's open a port on the VM for registry access:

```bash
kubectl patch service builds-registry --patch '{"spec":{"type":"NodePort","ports":[{"port":80,"nodePort":30500}]}}'
curl -I http://k3s.local:30500/v2/
```

If that curl worked we can probably build & push: `docker build -t k3s.local:30500/sample ./sample-app && docker push k3s.local:30500/sample`

### How does k8s pull them?

Nodes can, in all clusters I've tested, connect to Kubernetes Services' IPs.
They won't however resolve .cluster.local DNS names.
For this demo I'm instead exposing a NodePort, as seen above.

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample
  labels: &labels
    app: sample
spec:
  replicas: 3
  selector:
    matchLabels: *labels
  template:
    metadata:
      labels: *labels
    spec:
      containers:
      - name: hello
        image: 127.0.0.1:30500/sample:latest
        ports:
        - containerPort: 8080
EOF
```

## K3s ingress

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: sample
spec:
  selector:
    app: sample
  ports:
    - protocol: TCP
      port: 8080
EOF
cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: sample
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: sample
          servicePort: 8080
EOF
curl http://k3s.local/
```

## The development loop

I hadn't heard the term "development loop" until [this podcast episode](https://kubernetespodcast.com/episode/064-cloud-code/)
that distinguishes between "inner" and "outer" loop.

Let's try an inner loop with docker + kubectl ...

```
docker build -t k3s.local:30500/sample ./sample-app && docker push k3s.local:30500/sample
kubectl rollout restart deploy sample
curl http://k3s.local/
```

## K3s

Provision a volume.

Create an ingress.

What happens when I restart the cluster, not the VM?

Look at resource utilization.

# Things I'd like to explore

 * How can we run a k3os [iso](https://github.com/rancher/k3os/releases) on Mac with minimal overhead?
 * How to get `kubectl logs` to work with the docker-compose based provisioning?
