# Create fedora bare-metal cluster with kubeadm

Short-cut instructions that work for me, your mileage may vary.
Based on https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm

Setup is done via `ssh` from a "console" workstation, which should not
be part of the cluster. You need an SSH certificate on the console.

Console and servers need to be able to resolve each other's names.  I
used DHCP reservations on my router to get stable IP addresses and a
static /etc/hosts file for lookup.

Configuration files:
* config/ kubernetes config files
* etc/ config files for /etc on cluster servers
* save/ local copy of the original web guide for safe keeping.

You should review all the config files, you **must** modify:

* `./etc/hosts`
* `./config/metallb.yml`


From here on I assume that servers with names (or aliases) `master`,
`node1`, and `node2`. The environment variable HOSTS shoul be set to:

     HOSTS="node1 node2 master"

## Prepare the servers

Do a normal Fedora workstation install on each server, make sure to set the `hostname`.

On each server enable ssh and disable suspend if using laptops.

```
systemctl enable --now sshd
systemctl mask --now sleep.target suspend.target hibernate.target hybrid-sleep.target
```

The remaining steps are done from the console workstation via `ssh`.

Enable no-password ssh login for self and root. You'll have to enter
passwords only for this step. You need an SSH key pair in ~/.ssh on
the console workstation.

```
for h in $HOSTS; do
    ssh-copy-id $h
    ssh $h sudo -S cp -r $HOME/.ssh /root
done
```

Disable swapping (required by kubeadm)

```
for h in $HOSTS; do
    ssh root@$h swapoff -a # kubeadm wants no swapping
    ssh root@$h "sed -i -E 's/(^.*\\<swap\\>.*$)/# \\1/' /etc/fstab"
done
```

Turn off firewalls

```
for h in $HOSTS; do sudo systemctl mask firewalld; done
```

NOTE: It should be possible to open a limited set of ports but I haven't got that working yet. See: https://kubernetes.io/docs/setup/independent/install-kubeadm/#check-required-ports

## Cluster configuration

Copy configuration files from ./etc
```
for h in $HOSTS; do scp -r etc root@$h:/; done
```

Install packages (in background on all nodes concurrently - wait till finished!)
```
for h in $HOSTS; do ssh root@$h dnf install -y ipvsadm docker kubernetes-kubeadm & done
```

Enable basic services
```
for h in $HOSTS; do ssh root@$h systemctl enable --now docker kubelet; done
```

Initialize master, copy administrator config file to console so you can run kubectl.

Notes:
* --pod-network-cidr setting is for flannel below
* --ignore-preflight ignores spurious (I hope) "unsupported kernel version" warnings

```
ssh -lroot master kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=SystemVerification | tee kubeadm-init
scp root@master:/etc/kubernetes/admin.conf ~/.kube/config
```

Install the pod overlay network. I chose "flannel", other options are described at
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
kubectl node get -w # Wait till all nodes ready
```

Join your nodes to their master. Note the kubeadm-init file was generated
when you initialized the master above.

```
JOIN_CMD=$(grep 'kubeadm join' kubeadm-init)
for h in node1 node2; do ssh root@$h $JOIN_CMD --ignore-preflight-errors=SystemVerification; done
```

Install a load balancer. I picked metallb - https://metallb.universe.tf/

NOTE: you must upate config/metallb.yml for your network.

```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/metallb.yaml
kubectl apply -f config/metallb.yml
```

## Verifying the cluster

Deploy hello-world:
```
kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```

Expose hello-world as a NodePort - does not require a load balancer.
```
kubectl expose deployment hello-node --type=NodePort --port=8080
PORT=$(kubectl get svc hello-node -o=jsonpath='{.spec.ports[?(@.port==8080)].nodePort}')
curl master:$PORT
kubectl delete svc hello-node
```

Expose hello-world as a LoadBalancer - needs a load balancer configured.
```
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
kubectl get svc hello-node -w # Wait till it has an external IP
IP_ADDR=$(kubectl get svc hello-node -o=go-template='{{index .status.loadBalancer.ingress 0 "ip"}}')
curl $IP_ADDR:8080
kubectl delete svc hello-node
```

Run Sonobuoy compliance check from https://github.com/heptio/sonobuoy
No idea what it checks, but it passed so I'm Compliant!

```
go get github.com/heptio/sonobuoy

# Quick test set
sonobuoy run --wait --mode quick
sonobuoy e2e $(sonobuoy retrieve)
sonobuoy delete --wait --all

# Full test set
sonobuoy run --wait
sonobuoy e2e $(sonobuoy retrieve)
sonobuoy delete --wait --all
```

## Knative install

```
./bin/apply-knative
```

Verify istio: https://github.com/istio/istio/tree/master/samples/helloworld

## Re-setting the cluster

These steps attempt to completely wipe out all traces of an existing
cluster to avoid contaminating an attempt to build a new cluster.

This might not be necessary: `kubeadm reset` is supposed to clean up
all the effects of `kubeadm init`. Supposed to. I am not paranoid,
they really are out to get me.

Don't come crying to me if this turns your servers into paperweights.

```
for h in node1 node2; do
    kubectl drain $h --delete-local-data --force --ignore-daemonsets
    kubectl delete node $h
done
for h in $HOSTS; do
    ssh root@$h kubeadm reset --force
    ssh root@$h "dnf erase -y 'kube*' flannel etcd docker"
    ssh root@$h rm -rfv /etc/kubernetes /usr/libexec/kubernetes /etc/docker */var/lib/docker /usr/libexec/docker /etc/etcd /etc/sysconfig/flanneld /var/lib/cni
    ssh root@$h 'iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X'
done
for h in $HOSTS; do ssh root@$h reboot now; done
```

## Use a private docker registry

With a multi-node cluster you need to pull images from an external registry.

The popular minikube short-cut of using the cluster's docker daemon directly
only works because there is only on node, so only one docker daemon.

Here's how to use your own dockerhub.io account as a registry:
https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry

```
docker login # Adds credentails to ~/.docker/config.json
for h in $HOSTS; do scp ~/.docker/config.json root@$h:/var/lib/kubelet/config.json; done
# Now you can tag/pull using your dockerhub username.
```

## Other resources

These are guides similar to this one that I found later:
* https://unofficial-kubernetes.readthedocs.io/en/latest/getting-started-guides/kubeadm/
* https://developer.ibm.com/tutorials/developing-a-kubernetes-application-with-local-and-remote-clusters/
