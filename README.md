# Bonsai Kube: Rancher k3s, Rancher Server, Portainer and RKE cluster on Multipass VMs on your local machine, with MetalLB, HashiCorp Vault, Tekton Pipelines, vClusters and more addons


Latest version: Kubernetes 1.20

![multipass-k3s.png](multipass-k3s.png)

## Introduction

![bonsai](bonsai.jpg)

This repo provides a lightweight multi-node k3s deployment on multipass VMs on your local machine in step 1 and the deployment of a new full-fledged RKE cluster from Rancher Server running on k3s in step 2.

For a full-fledged RKE installation with rke tool on multipass VMs, please refer to:

https://github.com/arashkaffamanesh/multipass-rke-rancher

## About k3s

[k3s](https://k3s.io/) is a nice tool for development, fun and profit, with k3s you can spin up a lightweight k8s cluster from scratch in less than 3 minutes.

k3s is packed into a single binary, which already includes all you need to quickly spin up a k8s cluster.

To learn more about k3s, please visit [the k3s github repo](https://github.com/rancher/k3s) and [the official documentation.](https://rancher.com/docs/k3s/latest/en/)

## Prerequisites

### Install multipass (on MacOS or Linux)

You need Multipass running on your local machine, to learn more about Multipass, please visit:

https://github.com/CanonicalLtd/multipass

https://multipass.run/

This setup was tested on MacOS, but should work on Linux or Windows too.

You need to have about 4GB free RAM and 16GB free storage on your local machine, but it should work with less resources.

You need sudo rights on your machine.

You need kubectl in your path, if not, you can download the v1.17.4 version and put it in your path:

MacOS users:

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.4/bin/darwin/amd64/kubectl
```

Linux users:

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.4/bin/linux/amd64/kubectl
```

```bash
chmod +x ./kubectl
mv kubectl /usr/local/bin/
```

### Important hint for linux users

Linux users should adapt the `create-hosts.sh` and adapt the network interface name. You can find the nic name with:

```bash
multipass launch --name test
multipass exec test -- bash -c 'echo `ls /sys/class/net | grep en`'
```

Delete and purge the test VM:

```bash
multipass delete test
multipass purge
```

If the above doesn't work somehow, shell into the node and get the nic name:

```bash
multipass shell test
ifconfig
```

## k3s Deployment (step 1)

Clone this repo and deploy a 3-node deployment with a single command:

```bash
git clone https://github.com/arashkaffamanesh/multipass-k3s-rancher.git
cd multipass-k3s-rancher
./deploy-bonsai.sh
```

to enjoy the output of the total runtime:

############################################################################

Total runtime in minutes: 02:47

############################################################################

## What you get in step 1

You should get a running k3s cluster on 4 Multipass VMs with Rancher Server on top in about 4-5 minutes. Node1 is the master, all other 3 nodes are the workers.

## Accessing the Rancher Server on k3s

A tab in your browser should open after the deployment and point to:

https://k3s-worker1

Yo need to accept the self signed certificate, set the admin password and set the server url in Rancher GUI.

Note: we are using a self signed CA for node2 to be able to deploy a new RKE cluster from Rancher Server GUI running on k3s in the next step.

## Re-Deploy traefik with dashboard (optional)

If you'd like to redeploy traefik on k3s from scratch, you may want to run:

```bash
./4-deploy-traefik-dashboard.sh
```

## Deploy Metal-LB for Load Balancing

```bash
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml
kubectl create -f metal-lb-layer2-config.yaml
kubectl apply -f nginx.yaml
kubectl get svc
```

With that you have exposed nginx with the type LoadBalancer with an External-IP on your local machine! In my case the LoadBalancer External-IP was 192.168.64.24, which means you can access nginx with:

http://192.168.64.24 # with load balancing

or through ingress on any worker node:

https://k3s-worker2 # direct access through traefik

## RKE Deployment (step 2)

Now you can launch additional multipass rke VMs (rke1..3) and install docker on them with:

```bash
./5-deploy-3-multipass-rkes.sh
```

After the install is complete, add a new cluster via Rancher GUI, use flannel as networking, copy the provided command, shell into the 3 rke nodes and fire the command on all nodes. Your new RKE cluster should show up in Rancher GUI after few minutes in `Active` state.

```bash
multipass shell rke1
```

and run something like this:

```bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.2.8 --server https://node2 --token dkwwqh7mtkrml55sqvmtkr5xm6hbt2tx8l8vf95lgltvnd82wncs6z --ca-checksum da5882d7b45acb72325a2ce5e3b196481ce0f851c8e70fb9582f58a16b7d3f6d --etcd --controlplane --worker
```

Again, you'll get this command from Rancher GUI and you shall repeat the above step on rke2 and rke3 nodes as well.

In Rancher GUI you will see the rke nodes are getting registered and after few minutes the rke cluster state should change from `Provisioning / Updating` state to the `Active` state.

Grab the kubeconfig file from the GUI and save it as rke.yaml in the install directory and run:

```bash
export KUBECONFIG=rke.yaml
kubectl get nodes
kubectl config get-contexts
```

You should get something simliar to this:

```
multipass-k3s-rancher $ kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO     NAMESPACE
          rke        rke        user-qq2jg
*         rke-rke1   rke-rke1   user-qq2jg
          rke-rke2   rke-rke2   user-qq2jg
          rke-rke3   rke-rke3   user-qq2jg
```

## Accessing the RKE Cluster without Rancher Server

You can access the RKE cluster withouth the Rancher Server running on k3s, try the following by stopping the master node1 and access the RKE cluster through the rke-rke1 context:

```bash
multipass stop node1
kubectl --context rke-rke1 get nodes --kubeconfig=rke.yaml
```

After stopping node1, which is the k3s master / control plane, you'll see that you can't access the Rancher GUI on k3s and the context rke!

Note: start node1 again to enjoy Rancher running on k3s again:

```bash
multipass start node1
open https://node2
```

## Performance tip

If you'd need to increase the number of vCPUs, RAM and storage, you can adapt the `1-deploy-multipass-vms.sh` script and set e.g.:

```bash
--cpus 4 --mem 4G --disk 10G
```

## Add-Ons

### OpenEBS

If you'd like to have OpenEBS on your RKE cluster on your local machine, run:

```bash
./6-deploy-addon-openebs.sh
```
and enjoy the power of CNS (Cloud Native Storage), not only on your local machine!

### Prometheus operator

If you'd like to have Prometheus on your RKE cluster on your local machine, run:

```bash
./6-deploy-addon-prometheus.sh
```


## Clean Up

```bash
./cleanup.sh
./cleanup-rkes.sh
```

## Troubleshooting

k3s uses containerd as CRI (container runtime interface). If you'd like to see the status of the containers on the nodes for e.g. troubleshooting or fun, you can run:

```bash
multipass exec k3s-worker -- /bin/bash -c "sudo crictl ps -a"
```

or shell into the node and run `sudo crictl ps -a`:

```bash
multipass shell k3s-worker
sudo crictl ps -a
```

## Gotchas

Running `./cleanup-rkes.sh` throws an error like:

```bash
Stopping requested instances -[2019-09-14T14:49:51.096] [error] [rke1] process error occurred Crashed
```

which can be ignored.

## Credits

Thanks to the awesome folks at Rancher Labs for making k3s the first choice for a lightweight Kubernetes solution.

And thanks to Mattia Peri for his [great post](https://medium.com/@mattiaperi/kubernetes-cluster-with-k3s-and-multipass-7532361affa3) on medium, which encouraged me to automate everything with this small implementation for k3s on multipass.


## Blog post

A related blog post will is published here:

https://blog.kubernauts.io/k3s-with-metallb-on-multipass-vms-ac2b37298589?


## Related resources

[The Enterprise Grade Rancher Deployment Guide, the hard way](https://blog.kubernauts.io/enterprise-grade-rancher-deployment-guide-ubuntu-fd261e00994c)

[Announcing Maesh, a Lightweight and Simpler Service Mesh Made by the Traefik Team](https://blog.containo.us/announcing-maesh-a-lightweight-and-simpler-service-mesh-made-by-the-traefik-team-cb866edc6f29)

[Howto – Set up a highly available instance of Rancher](https://blog.ronnyvdb.net/2019/01/20/howto-set-up-a-highly-available-instance-of-rancher)

[Terraform configs and asnible playbooks to deploy k3s clusters](https://github.com/AnchorFree/ansible-k3s)

[OpenEBS](https://openebs.io)

[Cloud-Native stateful storage for Kubernetes with Rancher Labs' Longhorn](https://www.civo.com/learn/cloud-native-stateful-storage-for-kubernetes-with-rancher-labs-longhorn)

[Running k3s with metallb on Vagrant](https://medium.com/@toja/running-k3s-with-metallb-on-vagrant-bd9603a5113b)

[vagrant-k3s-metallb](https://github.com/otsuarez/vagrant-k3s-metallb)

[Kubernetes & Traefik 101 When Simplicity Matters](https://medium.com/@geraldcroes/kubernetes-traefik-101-when-simplicity-matters-957eeede2cf8)

[k3s + Gitlab](https://github.com/apk8s/k3s-gitlab)

[Using a k3s Kubernetes Cluster for Your GitLab Project](https://medium.com/better-programming/using-a-k3s-kubernetes-cluster-for-your-gitlab-project-b0b035c291a9)

[KIND and Load Balancing with MetalLB on Mac](https://www.thehumblelab.com/kind-and-metallb-on-mac/)

[Using MetalLB And Traefik Load Balancing For Your Bare Metal Kubernetes Cluster – Part 1](https://www.devtech101.com/2019/02/23/using-metallb-and-traefik-load-balancing-for-your-bare-metal-kubernetes-cluster-part-1/)
