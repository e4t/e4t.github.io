---
layout: post
title:  "Quick Install: Kubernets Cluster (RKE2) with Warewulf"
date:   
categories: Warewulf K8s RKE2 leap.
author: Egbert Eich
---

In a previous [blog
 post](https://e4t.github.io/warewulf/k8s/rke2/leap/2024/10/09/Deploy-RKE2-with-Warewulf.html)
 we've learned how we can leverage Warewulf - a node deployment tool used in
 High Performance Computing (HPC) - to deploy a K8s cluster using RKE2. The
 deployment was described step-by step explaining the rationale behind each
 step.  
This post supplements the previous post by providing a quick install guide
taking advantage of pre-build containers. With the help of these we are able
to perform the deployment of an RKE2 cluster with a few simple commands.  
Warewulf uses PXE boot to bring up the machines, so your cluster nodes need
to be configured for this and the network needs to be configure so that the
nodes will PXE-boot from the Warewulf deployment server. How to do this and
how to make node known to Warewulf is covered in a [post](https://mslacken.github.io/2024/03/26/install-wartewulf4.html)
about the basic Warewulf setup.

# Prerequisite: Prepare Configuration Overlay Template for RKE2
K8s agents and servers use a shared secret for authentication - the
connection token. Moreover, agents need to know the host name of the
server to contact.  
This information is provided in a config file: `/etc/rancher/rke2/config.yaml`
We let Warewulf provide this config file as a configuration overlay.
Since Warewulf does not come with a template for this overlay, we need
to create it ourselves which we will call `rke2-config`:
```
wwctl overlay create rke2-config
```
create a configuration template:
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
import it into the overlay and set its owner and permission:
```
wwctl overlay import --parents rke2-config /tmp/config.yaml.ww /etc/rancher/rke2/config.yaml.ww
wwctl overlay chown rke2-config /etc/rancher/rke2/config.yaml.ww 0
wwctl overlay chmod rke2-config /etc/rancher/rke2/config.yaml.ww 0600
```

# Set Up the Environment
First we set a few environment variables which we will use later.
```
cat > /tmp/rke2_environment.sh <<"EOF"
server=<add the server node here>
agents="<list of agents>"
container_url=docker://registry.opensuse.org/network/cluster/containers/containers-15.6
server_container=$container_url/warewulf-rke2-server:v1.30.5_rke2r1
agent_container=$container_url/warewulf-rke2-agent:v1.30.5_rke2r1
token="$(printf 'K'; \
         for n in {1..20}; do printf %x $RANDOM; done; \
         printf "::server:"; \
         for n in {1..20}; do printf %x $RANDOM; done)"
EOF
```
Here we assume we have one server node and multiple agent nodes whose
host names will have to replace in the above script.
Lists of nodes can be specified as a comma-seperated list, but also as
a range of nodes using square brackets (for example `k8s-agent[00-15]`
would refer to `k8s-agent00` to `k8s-agent15`) or lists and ranges combined.

Also, the script generates a connection token for the entire cluster. This
token is used to allow agents (and secondary servers) to connect to the
primary server. If we set up the server persistently outside of Warewulf
we either need to add this token to the config file `/etc/rancher/rke2/config.yaml`
on the server before we start the `rke2-server` service for the first
time. In case the server is already running - we need to grab the token
from it - from the file `/var/lib/rancher/rke2/server/node-token`.

Finally, we set the environment:
```
source /tmp/rke2_environment.sh
```
# Obtain the Node Images
The openSUSE registry has pre-built image for K8s agents. Taking advantage
of these removes the tedious task of preparing the base images. We pull
these and build the container:
```
wwctl container import ${agent_container} leap15.6-RKE2-agent
wwctl container build leap15.6-RKE2-agent

```

In the [previous blog](https://e4t.github.io/warewulf/k8s/rke2/leap/2024/10/09/Deploy-RKE2-with-Warewulf.html#server_considerations)
we've talked about the implications deploying K8s servers using Warewulf.
If we plan to deploy the server using Warewulf as well, we pull the server
container from the registry also  and build it:
```
wwctl container import ${server_container} leap15.6-RKE2-server
wwctl container build leap15.6-RKE2-server
```
# Set up Profiles
We utilize Warewulf profiles to configure different aspects of the setup
and assign these profiles to the different types of nodes as needed.
## Set up `rootfs`
The settings in this profile will ensure that a rootfs type is set so
that a call to `pivot_root()` performed by the container runtime will
succeed.
```
wwctl profile add container-host
wwctl profile set --root tmpfs -A "crashkernel=no net.ifnames=1 rootfstype=ramfs" container-host
```
## Set up Container Storage
Since the deployed nodes are ephemeral - that is they run out of a RAM disk,
we may want to to store container images as well as administrative data
on disk. Luckily, Warewulf is capable of configuring disks on the deployed
nodes.
For this example, we assumes that the container storage should reside on
disk `/dev/sdb`, we are using the entire disk and we want to scrap the data
at every boot.
Of course other setups are possible, check the [upstream
documentation](https://warewulf.org/docs/main/contents/disks.html) for
details.
```
wwctl profile add container-storage
wwctl profile set --diskname=/dev/sdb --diskwipe "true" \
      --partnumber 1 --partcreate "true" --partname container_storage \
      --fsname container_storage --fswipe "true" --fsformat ext4 \
	  --fspath /var/lib/rancher \
      container-storage
```
Note: When booting the node from Warewulf, the entire disk (`/dev/sdb` in
this example) will be wiped. Make sure it does not contain any valuable
data!

## Set up the Connection Key
When we've set up our environment, we generated a unique connection key which
we will use throughout this K8s cluster to allow agents (and secondary servers)
to connect to the primary server.
```
wwctl profile add rke2-config-key
wwctl profile set --tagadd="connectiontoken=${token}" \
              -O rke2-config rke2-config-key
```
## Set up the Server Profile
Here, we set up a profile which is used to point agents - and secondary
servers - to the primary server:
```
wwctl profile add rke2-config-first-server
wwctl profile set --tagadd="server=${server}" -O rke2-config rke2-config-first-server
```
# Set up and Start the Nodes
Now, we are ready to deploy our nodes.
## Set up and deploy the Server Node
If applicable, set up the server node
```
wwctl node set \
      -P default,container-host,container-storage,rke2-config-key \
      -C leap15.6-RKE2-server ${server}
wwctl overlay build ${server}
```
Now, we are ready to PXE-boot the first server.
## Set up and deploy the Agent Nodes
We now configure the agent nodes and build the overlays:
```
wwctl node set \
      -P default,container-host,container-storage,rke2-config-key,rke2-config-first-server \
      -C leap15.6-RKE2-agent ${agents}
wwctl overlay build ${agents}
```
Once the first server node is up and running we can now start PXE-booting
the agents. They should connect to the server automatically.

# Check Node Status
To check the status of the nodes, we connect to the server through
ssh and run
```
kubectl get nodes
```
to check the status of the available agents.
