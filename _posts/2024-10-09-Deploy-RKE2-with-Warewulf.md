---
layout: post
title:  "Deploy a Kubernets Cluster (RKE2) with Warewulf"
date:   2024-10-09
categories: Warewulf K8s RKE2 leap
author: Egbert Eich
---

In High Performance Computing (HPC) we frequently encounter node counts
in compute clusters that are impractical to be managed manually.
Here, the saving grace is that the number of variations in installation
and configuration among nodes of a cluster is small. Also, the
number of parameters that are individual to each node is low.
Thus, in the 'cattle/pet' model, compute nodes would be treated like cattle.  
Warewulf, a deployment system for HPC compute nodes, is specifically
designed for this case. It utilizes PXE to boot nodes and
provide their root filesystem. Nodes are ephemeral, i.e. their root
filesystem resides in a RAM disk. In a recent [Blog
post](https://mslacken.github.io/2024/03/26/install-wartewulf4.html),
Christian Goll described how to set up and manage a cluster using Warewulf.  
Kubernetes (K8s) deployments potentially face similar challanges:
K8s clusters often consist of a large number of mostly identical agent
nodes with a minimal installation and very little individual configuration.  
In this article we explore how to set up a K8s cluster with Rancher's
next-generation Kubernetes distribution RKE2 using Warewulf.

# Considerations

## K8s Server

In K8s we distinguish between a 'server' and 'agents'. While a 'server'
may act as an agent as well, it is mainly to organize and control the
cluster. In a sense it is comparable to a 'head node' in HPC.
It is possible to deploy the server role using Warewulf - and we have
done so for our experiments. However, at present Warewulf is capable of
deploying ephemeral systems only while the server role may require to
maintain some state. Therefore, it may be preferrable to set it up as
a permanent installation and utilize Warewulf for agent depoyment only.
We will still describe how to deploy a server using Warewulf.

## Container Image Storage

Since our workloads are containerized, the container host requires only
a very minimal installation. This installation - together with RKE2 -
will not use up much of a node's memory when running out of a RAM disk.
This is different for container images which are pulled from registries
and stored locally. If these were stored on RAM disk, memory would quickly
be exhausted. Fortunately, warewulf is able to set up mass storage devices -
optionally every time a node is started. We will show how to set up
storage for container images using Warewulf.

## Basic Setup

This post will not cover how to perform the basic network setup required
for the nodes to PXE-boot from the Warewulf deployment server or make
nodes known to Warewulf. These topocs are all covered in [Christian's
Blog](https://mslacken.github.io/2024/03/26/install-wartewulf4.html)
already.

# Setup

## Create Deployment Image
Warewulf utilizes container registries to obtain installation images.
We start by importing a base container image from the openSUSE
registry.
```
wwctl container import \
  docker://registry.opensuse.org/science/warewulf/leap-15.6/containers/kernel:latest \
  leap15.6-RKE2
```
### General Image Preparation
Since this base image is generic, we need to install any missing packages
required to install and start the RKE2 service. First, open up a shell
inside the node image:
```
wwctl container shell leap15.6-RKE2
```
and run:
```
zypper -n in -y tar iptables awk
cd /root
curl -o rke2.sh -fsSL https://get.rke2.io
```
`tar` and `awk` are required by the RKE2 install script while `iptables`
is required by K8s to set up the container network.

### Image Preparation for Container Image Storage
This step is optional, but it is advisable to set up a storage device to
hold container images. Container image storage is required on every node
that will act as an agent - including the server node.  
First we need to prepare the deployment image. To do so, we log into the
image again to we create the image directory:
```
mkdir /var/lib/rancher
```
Then we install the packages required to perform the setup:
```
zypper -n in -y --no-recommends ignition gptfdisk
```
### Prepare the Image for the K8s Server
Now, we are done with the common setup. We can exit the shell session
in the container. When doing so, we need to make sure, the container
image is rebuilt. We should see a message:
`Rebuilding container...`. If this is not the case, we need to rebuild the
image by hand:
```
wwctl container build leap15.6-RKE2
```
It's recommend to install the K8s Server permanently, therefore, if
we would like to follow this recommendation, we can skip the reminder
of this section.

Otherwise, we clone our image for the server:
```
wwctl container copy leap15.6-RKE2 leap15.6-RKE2-server
```
open a shell in the newly created server image:
```
wwctl container shell leap15.6-RKE2-server
```
install and enable `rke2-server` and adjust the environment for RKE2:
```
# Install the RKE2 tarball and prepare for server start
cd /root
INSTALL_RKE2_SKIP_RELOAD=true INSTALL_RKE2_VERSION="v1.31.1+rke2r1" sh rke2.sh
# Enable the service so it comes up later
systemctl enable rke2-server
# For container deployment we want `helm`
zypper -n in -y helm
# Set up environment so kubectl and crictl are found and will run
cat > /etc/profile.d/rke2.sh << EOF
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
EOF
```
We are pinning the version to the one this has been tested
with. If we omit `INSTALL_RKE2_VERSION=...` we will get the
[latest version](https://github.com/rancher/rke2/releases).
Now, we exit the shell in the server container and again make sure the
image is rebuilt.


### Prepare the Image for the K8s Agents
We need to finalize the agent image by downloading and installing
RKE2 and enabling the `rke2-agent` service. For this, we log
into the container
```
wwctl container shell leap15.6-RKE2
```
and run:
```
cd /root
INSTALL_RKE2_SKIP_RELOAD=true INSTALL_RKE2_TYPE="agent" \
 INSTALL_RKE2_VERSION="v1.31.1+rke2r1" sh rke2.sh
systemctl enable rke2-agent
```
Here, we are pinning the RKE2 version to the version this has
been tested with. This has to match the version of the server
node. If the server node has not been deployed using Warewulf,
we need to make sure its version matches the version used here.
If we omit `INSTALL_RKE2_VERSION=...` we will get the
[latest version](https://github.com/rancher/rke2/releases).
When logging out, we make sure, the container image is rebuilt.

## Set up a Configuration Template for RKE2
Since the K8s agents and servers need a shared secret - the connection
token - and secondary nodes need information about the primary server
to connect, we set up a warewulf configuration overlay template for
these.  
We vreate a new overlay `rke2-config` on the Warewulf deployment
server by running:
```
wwctl overlay create rke2-config
```
create a configuration template
<!-- {% raw %} -->
```
cat > /tmp/config.yaml.ww <<EOF
{{ if ne (index .Tags "server") "" -}}
server: https://{{ index .Tags "server" }}:9345
{{ end -}}
{{ if ne (index .Tags "clienttoken") "" -}}
token: {{ index .Tags "connectiontoken" }}
{{ end -}}
EOF
```
<!-- {% endraw %} -->
and import it into the overlay setting its owner and permission:
```
wwctl overlay import --parents rke2-agent /tmp/config.yaml.ww /etc/rancher/rke2/config.yaml.ww
wwctl overlay chown rke2-agent /etc/rancher/rke2/config.yaml.ww 0
wwctl overlay chmod rke2-agent /etc/rancher/rke2/config.yaml.ww 0600
```
This template will create a `server:` entry pointing to the
communication endpoint (address & port) of the primary K8s server
and a `token:` which will hold the client token in case
these entries exist in the configuration of the node or
one of its profiles. (These templates use the [Golang text/template
engine](https://pkg.go.dev/text/template). Also, check the
[upstream documentation](https://warewulf.org/docs/main/contents/templating.html)
for the template file syntax.)  

## Set up Profiles
At this point, we create some profiles which we will use for setting
up all node, i.e. the Agents and - if  applicable - the Server. To
simplify things, we assume the hardware for all the nodes is identical.

### The 'Switch to `tmpfs`' Profile
Container runtimes require `pivot_root()` to work, which is not possible
as long as we are still running out of a rootfs. This is not only the case
for K8s but also for podman. Since the default init process in a Warewulf
deployment doesn't perform a `switch_root`, we need to change this.
To do so, we need to perform two things:
1. Make sure that rootfs is not a tmpfs. This can be done by adding
   `rootfstype=ramfs` to the kernel command line.
2. Let `init` know that we intend to switch to tmpfs.
We do this by setting up a profile for container hosts:
```
wwctl profile add container-host
wwctl profile set --root=tmpfs -A "crashkernel=no net.ifnames=1 rootfstype=ramfs" container-host
```
(Here, `crashkernel=no net.ifnames=1` are the default kernel arguments.)

### Set up the Container Storage Profile
As stated above, this step is optional but recommended.  
To set up storage on the nodes, the deployment images need to be
prepared as describe above in section 'Image Preparation for
Container Image Storage'.  
For simplicity, we assume that all nodes will receive the identical
storage configuration. Therefore, we create a profile which we
will add to the nodes later. It, however, would be easy to set up
multiple profiles or override settings per node.  
We create the profile `container-storage` and set up the disk, partition,
file system and mount point:
```
wwctl profile add container-storage
wwctl profile set --diskname <disk> --diskwipe[=false] \
   --partname container_storage --partnumber 1  --partcreate=true \
   --fsname container_storage --fsformat ext4 --fspath /var/lib/rancher
   container-storage
```
Here, we need to replace `<disk>` by the physical storage device we want to
use. If the disks are not empty initially, we should set the option
`--diskwipe=true`. This will cause the disks to be wiped on every
consecutive boot, therefore, we may want to unset this later.
`--partcreate` makes sure, the partition is created
if it doesn't exist. Most other arguments should be self-explanatory.
If we need to set up the machines multiple times and want to make sure
the disks are wiped each time, we should not rely on the `--diskwipe`
option which in fact only wipes the partition table: if an identical
partion table is recreated, `ignition` will not notice and reuse the
partition from a previous setup.

### Set up the Connection Token Profile
RKE2 allows to configure a connection token to both Servers and Agents.
If none is provided to the primary server it will be generated internally.
 If we set up the server persistently, we need to create a file
 `/etc/rancher/rke2/config.yaml` with the content:
```
token: <connection_token>
```
before we start this server for the first time, or if the server has been
started before already, we need to obtain the token from the file
`/var/lib/rancher/rke2/server/node-token` on this machine and use it
for the `token` variable below.
We now run:
```
wwctl profile add rke2-config-key
```
generate the token, add the `rke2-config` overlay to the profile and
set a tag containing the token that will later be used by the profile:
```
token="$(printf 'K'; \
         for n in {1..20}; do printf %x $RANDOM; done; \
         printf "::server:"; \
         for n in {1..20}; do printf %x $RANDOM; done)"
wwctl profile set --tagadd="connectiontoken=${token}" \
              -O rke2-config rke2-config-key
```
### Set up the 'First Server' Profile
This profile is used to point the agents (and secondary servers)
to the initial server:
```
wwctl profile add rke2-config-first-server
wwctl profile set --tagadd="server=${server}" -O rke2-config rke2-config-first-server
```

## Start the Nodes
With these profiles in place, we are now able to set up and boot all
machine roles.

## Start and Test the first K8s Server
If we use Warewulf to also deploy the K8s server, we need to start it now and
make sure it is running correctly before we proceed to start the nodes.
Otherwise, we assume a server is running already which we can connect
via ssh and proceed to the next section.

It's assumed that we have already performed a basic setup of the server
node (like make its MAC and designated IP address known to Warewulf).
First we add the configuration profiles to the server. This includes
the `container-host` and `container-storage` as well as the `rke2-config-key`
profiles. We also set the container image:
```
wwctl node set -P default,container-host,container-storage,rke2-config-key -C leap15.6-RKE2-server <server_node>
```
Finally, we build the overlays:
```
wwctl overlay build <server_node>
```

Now, we are ready to power on the server and wait until is has booted. Once
this is the case, We log into it via `ssh`. There we can observe the RKE2 server
service starting:
```
systemctl status rke2-server
```
The output will show `containerd`, `kubelet` and several instances of runc
(`containerd-shim-runc-v2`) running. When the initial containers have
completed starting, the output should contain the lines:
```
Oct 07 16:36:36 dell04 rke2[1299]: time="2024-10-07T16:36:36Z" level=info
msg="Labels and annotations have been set successfully on node: k8s-server"
Oct 07 16:36:42 dell04 rke2[1299]: time="2024-10-07T16:36:42Z" level=info msg="Adding node k8s-sesrver-d034de85 etcd status condition"
Oct 07 16:37:00 dell04 rke2[1299]: time="2024-10-07T16:37:00Z" level=info msg="Tunnel authorizer set Kubelet Port 0.0.0.0:10250"
```
We can watch the remaining services starting by running:
```
kubectl get pods -A
```
Once all services are up and running, the output should look like this:
```
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS   AGE
kube-system   cloud-controller-manager-k8s-server                     1/1     Running     0          20m
kube-system   etcd-k8s-server                                         1/1     Running     0          19m
kube-system   helm-install-rke2-canal-lnvv2                           0/1     Completed   0          20m
kube-system   helm-install-rke2-coredns-rjd54                         0/1     Completed   0          20m
kube-system   helm-install-rke2-ingress-nginx-97rh7                   0/1     Completed   0          20m
kube-system   helm-install-rke2-metrics-server-8z878                  0/1     Completed   0          20m
kube-system   helm-install-rke2-snapshot-controller-crd-mt2ds         0/1     Completed   0          20m
kube-system   helm-install-rke2-snapshot-controller-l5bbp             0/1     Completed   0          20m
kube-system   helm-install-rke2-snapshot-validation-webhook-glkgm     0/1     Completed   0          20m
kube-system   kube-apiserver-k8s-server                               1/1     Running     0          20m
kube-system   kube-controller-manager-k8s-server                      1/1     Running     0          20m
kube-system   kube-proxy-k8s-server                                   1/1     Running     0          20m
kube-system   kube-scheduler-k8s-server                               1/1     Running     0          20m
kube-system   rke2-canal-xfq6l                                        2/2     Running     0          20m
kube-system   rke2-coredns-rke2-coredns-6bb85f9dd8-fj4r4              1/1     Running     0          20m
kube-system   rke2-coredns-rke2-coredns-autoscaler-7b9c797d64-rxkmm   1/1     Running     0          20m
kube-system   rke2-ingress-nginx-controller-nmlhg                     1/1     Running     0          19m
kube-system   rke2-metrics-server-868fc8795f-gz6pz                    1/1     Running     0          19m
kube-system   rke2-snapshot-controller-7dcf5d5b46-8lp8w               1/1     Running     0          19m
kube-system   rke2-snapshot-validation-webhook-bf7bbd6fc-p6mf9        1/1     Running     0          19m
```
This server is now ready to accept agents (and secondary servers). If we
require additional servers for redundancy, their setup is identical, however,
we will need to add the `rke2-config-first-server` profile when setting up
the node above.

## Start and verify the Agent
Now, we are ready to bring up the agents. First, we set up the nodes by
adding the profiles `container-host`, `container-storage`,
`rke2-config-key` and `rke2-config-first-server` to all the client nodes:
```
agents=<agent_nodes>
wwctl node set -P default,container-host,rke2-agent,container-storage $agents
```
as well as the container image for the agent:
```
wwctl node set -C leap15.6-RKE2 <agent_nodes>
```
and rebuild the overlays for all agent nodes:
```
wwctl overlay build $agents
```
We replace `<agent_nodes>` by the  appropriate node names.
This can be a comma-seperated list, but also a range of nodes
specified in squuare brackets - for example `k8s-agent[00-15]` would
refer to `k8s-agent00` to `k8s-agent15` - or lists and ranges combined.

At this point, we are able to boot the first agent node. Once the first node
is up, we may log in using `ssh` and check the status of the `rke2-agent`
service:
```
systemctl status rke2-agent
```
The output should contain lines like:
```
Oct 07 19:23:59 k8s-agent01 rke2[1301]: time="2024-10-07T19:23:59Z" level=info msg="rke2 agent is up and running"
Oct 07 19:23:59 k8s-agent01 systemd[1]: Started Rancher Kubernetes Engine v2 (agent).
Oct 07 19:24:25 k8s-agent01 rke2[1301]: time="2024-10-07T19:24:25Z" level=info
msg="Tunnel authorizer set Kubelet Port 0.0.0.0:10250"
```
This should be all we check on the agent. Any further verifications will be
done from the server. We log into the server and run:
```
kubectl get nodes
```
This should produce an output like:
```
kubectl get nodes -A
NAME          STATUS   ROLES                       AGE    VERSION
k8s-server    Ready    control-plane,etcd,master   168m   v1.30.4+rke2r1
k8s-agent01   Ready    <none>                      69s    v1.30.4+rke2r1
```
We see that the first agent node is available in the cluster. Now, we can
spin up more nodes and repeat the last step to verify they appear.

# Conclusions
We've  shown that it is possible to deploy a functional K8s cluster with
RKE2 using Warewulf. We could for example proceed deploying the NVIDIA GPU
operator with a driver container on this cluster as described in a [previous
Blog](https://e4t.github.io/nvidia/cuda/k8s/rke2/nvidia-gpu-operator/leap/2024/09/24/NVIDIA-GPU-Operator-on-oS-Leap.html)
and set up a K8s cluster for AI workloads. Most of the steps were straight
forward and could be derived from the [Warewulf User
Guide](https://warewulf.org/docs/main/index.html). The only non-obvious
step to take were the ones required to set up the rootfs in a way that it
is ensured the container runtime is able to call `pivot_root`.
