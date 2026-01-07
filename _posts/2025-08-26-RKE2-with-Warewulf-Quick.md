---
layout: post
title:  "Quick Install: Kubernets Cluster (RKE2) with Warewulf"
date:   2025-08-26
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
A suitable overlay template can be installed from the package
`warewulf4-overlay-rk2`:
```
 zypper -n install -y warewulf4-overlay-rke2
```

# Set Up the Environment
First we set a few environment variables which we will use later.
```
cat > /tmp/rke2_environment.sh <<"EOF"
server=<add the server node here>
agents="<list of agents>"
container_url=docker://registry.opensuse.org/network/cluster/containers/containers-15.6
server_container=$container_url/warewulf-rke2-server:v1.32.7_rke2r1
agent_container=$container_url/warewulf-rke2-agent:v1.32.7_rke2r1
token="$(for n in {1..41}; do printf %2.2x $(($RANDOM & 0xff));done)"
EOF
```
Here we assume we have one server node and multiple agent nodes whose
host names will have to replace in the above script.
Lists of nodes can be specified as a comma-seperated list, but also as
a range of nodes using square brackets (for example `k8s-agent[00-15]`
would refer to `k8s-agent00` to `k8s-agent15`) or lists and ranges combined.

Also, the script generates a connection token for the entire cluster. This
token is used to allow agents (and secondary servers) to connect to the
primary server[^1]. If we set up the server persistently outside of Warewulf
we either need to add this token to the config file `/etc/rancher/rke2/config.yaml`
on the server before we start the `rke2-server` service for the first time
or grab it from the server once it has booted.  
In this example, we are using a 'short token' which does not contain
the fingerprint of the root CA of the cluster. For production environments
you are encouraged to create certificates for your K8s cluster beforehand
and calculate the fingerprint from the root CA for use in the agent token.
Refer to the [appendix below](#appendix-1) how to do so.  
In case your server is already running you will have to grab the token from
it instead and set the variable `token` in the script above accordingly.
It can be found in the file `/var/lib/rancher/rke2/server/node-token`.


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
wwctl profile set --diskname /dev/sdb --diskwipe \
      --partnumber 1 --partcreate --partname container_storage \
      --fsname container_storage --fswipe --fsformat ext4 \
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
      --image leap15.6-RKE2-server ${server}
wwctl overlay build ${server}
```
Now, we are ready to PXE-boot the first server.
## Set up and deploy the Agent Nodes
We now configure the agent nodes and build the overlays:
```
wwctl node set \
      -P default,container-host,container-storage,rke2-config-key,rke2-config-first-server \
      --image leap15.6-RKE2-agent ${agents}
wwctl overlay build ${agents}
```

# Start Up Nodes
Now, we are ready to fire up our nodes. We assume that all nodes are
presently turned off and configured to boot from PXE by default.
First, turn on and boot the server:
```
wwctl power on ${server}
```
Once the first server node is up and running we can now start PXE-booting
the agents.
```
wwctl power on ${agents}
```
They should connect to the server automatically.

# Check Node Status
To check the status of the nodes, we connect to the server through
ssh and run
```
kubectl get nodes
```
to check the status of the available agents.

# <a id="appendix-1"></a>Appendix: Set Up CA Certificates and caluculate Node Access Token
Rancher provides a script (`enerate-custom-ca-certs.sh`) to generate the
different certificates required for a K8s cluster.
First, download this script to current directory:
```
curl -sOL --output-dir
. https://github.com/k3s-io/k3s/raw/master/contrib/util/generate-custom-ca-certs.sh
```
and run the following commands:
```
export DATA_DIR=$(mktemp -d $(pwd)/tmp-XXXXXXXX)
chmod go-rwx $DATA_DIR
./generate-custom-ca-certs.sh
wwctl overlay rm -f rke2-ca
wwctl overlay create rke2-ca
files="server-ca.crt server-ca.key client-ca.crt client-ca.key \
	request-header-ca.crt request-header-ca.key \
    etcd/peer-ca.crt etcd/peer-ca.key etcd/server-ca.crt etcd/server-ca.key"
for d in $files; do
	d=${DATA_DIR}/server/tls/$d
    wwctl overlay import --parents rke2-ca $d /var/lib/rancher/rke2/${d#${DATA_DIR}/}.ww
    wwctl overlay chown rke2-ca /var/lib/rancher/rke2/${d#${DATA_DIR}/}.ww 0
    wwctl overlay chmod rke2-ca /var/lib/rancher/rke2/${d#${DATA_DIR}/}.ww $(stat -c %a $d)
done
ca_hash=$(openssl x509 -in $DATA_DIR/server/tls/root-ca.crt -outform DER \
	|sha256sum)
token="$( printf "K10${ca_hash%  *}::server:"; \
         for n in {1..41}; do printf %2.2x $(($RANDOM & 0xff));done )"
```
The certificates will be passed to the K8s in the system overlay.
If you are setting up mulitple K8s clusters you may want to create
separate certificates for each cluster and place the overlay name
into the clusters namespace instead of using `rke2-ca`.  
Before you delete `$DATA_DIR`, you may want to save the root and
and intermiate certificates and keys (`$DATADIR/server/tls/root-ca.*`,
(`$DATADIR/server/tls/intermediate-ca.*`) to a safe location.
If you have a root and intermediate CA you want to reuse, you may copy
these into the directory `$DATADIR/server/tls` before running
`generate-custom-ca-certs.sh`. Use `$token` as the agent token instead.

[^1]: Secondary servers need to find the primary server endpoint like nodes. Therefore, they recieve the access token like nodes.
