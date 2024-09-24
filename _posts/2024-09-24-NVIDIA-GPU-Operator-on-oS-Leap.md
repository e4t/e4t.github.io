---
layout: post
title:  "Installing the NVIDIA GPU Operator on Kubernetes on openSUSE Leap"
date:   2024-09-24
categories: nvidia cuda K8s RKE2 nvidia-gpu-operator leap
author: Egbert Eich
---

This article shows how to install and deploy Kubernetes (K8s) using
RKE2 by SUSE Rancher on openSUSE Leap 15.6 with the NVIDIA GPU Operator.
This operator deploys and loads any driver stack components required by
CUDA on K8s Cluster nodes without touching the container host and makes
sure, the correct driver stack is made available to driver containers.
We use a driver container specifically build for openSUSE Leap 15.6 and
SLE 15 SP6.
GPU acceleration with CUDA is used in many AI applications. AI application
workflows are frequently depoyed through K8s.

# Introduction

NVIDIA's Compute Unified Device Architecture (CUDA) plays a crucial role in
AI today. Only with the enormous compute power of state-of-the-art GPUs it
is possible to process training and inferencing with an acceptable amount of
resources and compute time.

Most AI workflows rely on containerized workloads deployed and managed by
Kubernetes (K8s). To deploy the entire compute stack - including kernel
modules - to a K8s cluster, NVIDIA has designed its GPU Operator, which,
together with a set of containers, is able to perform this task without
ever touching the container hosts.

Most of the components used by the GPU Operator are 'distribution agnostic'
however, one container needs to be built specifically for the target
distribution: the driver container. This is owed to the fact that drivers
are loaded into the kernel space and therefore need to be built specifically
for that kernel.

For a long time, NVIDIA kernel drivers were proprietary and closed source.
More recently, NVIDIA has published a kernel driver that's entirely open
source. This enables Linux distributions to publish pre-built drivers
for their products. This allows for a much quicker installation. Also,
prebuilt drivers are signed with the key thats used for the distribution
kernel. This way, the driver will work seamlessly in systems with secure
boot enabled. The container utilized below makes use of a pre-built driver.

In the next section we will explore how to deploy K8s on openSUSE Leap 15.6
once this is done, we will deploy the NVIDA GPU Operator in the following
section run some initial tests. If you have K8s already running you may want
to skip ahead to the 2nd part.

# Install RKE2 on openSUSE Leap 15.6

We have chosen RKE2 from SUSE Rancher for K8s over the K8s packages
shipped with openSUSE Leap: RKE2 is a well curated and maintained
Kubernetes distribution which works right out of the box while
openSUSE's K8s packages have been broken pretty much ever since
openSUSE Kubic has been dropped.

RKE2 does not come as an RPM package. This seems strange at first,
however, it is owed to the fact that Rancher wants to ensure maximal
portability across various Linux distributions.

Instead, it comes as a tar-ball - which is not unusual for application
layer software.

Most of what's described in this document has been taken from a great
article by Alex Arnoldy on how to deploy NVIDIA's GPU Operator on RKE2
and SLE BCI. Unfortunately, it was no longer fully up-to-date and thus
has been taken down.

## Install the K8s server

Kubernetes consists of at least one server which serves as a control
node for the entire cluster. Additionally clusters may have any number
of agents - i.e. machines which workloads will be spread across.
Servers will act as an agent as well. If your K8s cluster consists
just of one machine, you will be done once your server is installed.
You may skip the following section.
For system requirements you may want to check [here](https://docs.rke2.io/install/requirements).
We assume, you have a Leap 15.6 system  installed already (minimal
installation is sufficient and even preferred).

1. Make sure, you have all components installed already  which are either
   required for installation or runtime:
   ```
   zypper -n install -y curl tar gawk iptables helm
   ```
   For the installation, a convenient installation script exists. This
   downloads the required components, performs a checksum verification
   and installs them. The installation is minimal. When RKE2 is started
   for the first time, it will install itself to `/var/lib/rancher` and
   `/etc/rancher`.
   Download the installation script:
   ```
   # cd /root
   # curl -o rke2.sh -fsSL https://get.rke2.io
   ```
2. and run it:
   ```
   sh rke2.sh
   ```
3. To make sure, that the binaries provided by RKE2 - most importantly,
   `kubectl` - are found and will find their config files, you may want
   to create a separate shell profile:
   ```
   #  cat > /etc/profile.d/rke2.sh << EOF
   export PATH=$PATH:/var/lib/rancher/rke2/bin
   export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
   export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
   EOF
   ```
4. Now enable and start the `rke2-server` service:
   ```
   systemctl enable --now rke2-server
   ```
   With this, the installation is completed.
5. To check is all pods have come up properly and are running of have
   completed successfully, run:
   ```
   # kubectl get nodes -n kube-system
   ```

## Install Agents

If you are running a single node cluster, you are done now and may skip this
chapter. Otherwise, you will need to perform the steps below for every node
you want to install as an agent.
1. As above, make sure, all required prerequisites are installed:
   ```
   # zypper -n install -y curl tar gawk iptables
   ```
2. Download the installation script
   ```
   # cd /root
   # curl -o rke2.sh -fsSL https://get.rke2.io
   ```
3. and run it:
   ```
   # INSTALL_RKE2_TYPE="agent" sh rke2.sh
   ```
4. Obtain the token from the server node, it can be found on the server
   at `/var/lib/rancher/rke2/server/node-token`.
   and add it to config file for the RK2 agent service:
   ```
   # mkdir -p /etc/rancher/rke2/
   # cat > /etc/rancher/rke2/config.yaml
   server: https://<server>:9345
   token <obtained token>
   ```
   (You have to replace <server> by the name of IP of the RKE2 server host
   and <obtained token> by the agent token mentioned above.
5. Now you are able to start the agent:
   ```
   kubectl enable --now rke2-agent
   ```
6. After a while you should see that the node is has been picked up
   by the server. Run:
   ```
   kubectl get nodes
   ```
   in the server machine. The output should look something like this:
   ```
   NAME     STATUS   ROLES                       AGE    VERSION
   node01   Ready    control-plane,etcd,master   12m   v1.30.4+rke2r1
   node02   Ready    <none>                      5m    v1.30.4+rke2r1
   ```

# Deploying the GPU Operator

Now, with the K8s cluster (hopefully) running, you'd be ready to deploy the
GPU operator. The following steps need to be performed on the server node
only, regardless if this has a GPU installed or not. The correct driver
will be installed on any node that has a GPU installed.
1. To simply configuration, create a file `/root/build-variables.sh` on the
   server node:
   ```
   # cat > /root/build-variables.sh <<EOF
   export LEAP_MAJ="15"
   export LEAP_MIN="6"
   export DRIVER_VERSION="555.42.06"
   export OPERATOR_VERSION="v24.6.1"
   export DRIVER_IMAGE=nvidia-driver-container
   export REGISTRY="registry.opensuse.org/network/cluster/containers/containers-${LEAP_MAJ}.${LEAP_MIN}"
   EOF
   ```
2. and source this file from the shell you run the following commands from:
   ```
   # source /root/build-variables.sh
   ```
   Note that in the script above we are using kernel driver version 555.42.06
   for CUDA 12.5 instead of CUDA 12.6 as in 12.6 NVIDIA has introduced some
   dependency issues which have not been resolved fully, yet. This will limit
   CUDA used in the payload to 12.5 or older since a kernel driver version will
   only work for CUDA versions older or equal to the version it was provided
   with.
   This will be fixed in future versions so that later driver of GPU operator
   versions can be used.
   Also note, that ``$REGISTRY`` points to a driver container in
   https://build.opensuse.org/package/show/network:cluster:containers/nv-driver-container
   This is a driver container specifically built for Leap 15.6 and SLE 15 SP6.
   The `nvidia-driver-ctr` container will look for a container image
   `${REGISTRY}/${DRIVER_IMAGE}` tagged: `${DRIVER_VERSION}-${ID}${VERSION_ID}`.
   `${ID}` and `${VERSION_ID}` are taken from `/etc/os-release` on the
   container host. Currently, the container above is tagged for Leap 15.6 and
   SLE 15 SP6.
3. Add the NVIDIA Helm repository:
   ```
   # helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
   ```
4. and update it:
   ```
   # helm repo update
   ```
5. Now deploy the operator using the nvidia/gpu-operator Helm chart:
   ```
   # helm install -n gpu-operator \
     --generate-name   --wait \
     --create-namespace \
     --version=${OPERATOR_VERSION} \
     nvidia/gpu-operator \
     --set driver.repository=${REGISTRY} \
     --set driver.image=${DRIVER_IMAGE} \
     --set driver.version=${DRIVER_VERSION} \
     --set operator.defaultRuntime=containerd \
     --set toolkit.env[0].name=CONTAINERD_CONFIG \
     --set toolkit.env[0].value=/var/lib/rancher/rke2/agent/etc/containerd/config.toml \
     --set toolkit.env[1].name=CONTAINERD_SOCKET  \
     --set toolkit.env[1].value=/run/k3s/containerd/containerd.sock \
     --set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS \
     --set toolkit.env[2].value=nvidia \
     --set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT \
     --set-string toolkit.env[3].value=true
    ```
   After a while, the command will return.
6. Now, you can view the additional pods that have started in the
   `gpu-operator` namespace:
   ```
   kubectl get pods --namespace gpu-operator
   ```
7. To verify that everything has been deployed correctly, run:
   ```
   # kubectl logs -n gpu-operator -l app=nvidia-operator-validator
   ```
   This should return a result like:
   ```
   Defaulted container "nvidia-operator-validator" out of: nvidia-operator-validator, driver-validation (init), toolkit-validation (init), cuda-validation (init), plugin-validation (init)
   all validations are successful
   ```
   Also, run:
   ```
   # kubectl logs -n gpu-operator -l app=nvidia-cuda-validator
   ```
   which should result in:
   ```
   Defaulted container "nvidia-cuda-validator" out of: nvidia-cuda-validator, cuda-validation (init)
   cuda workload validation is successful
   ```
   To obtain information on the NVIDIA hardware installed on each node, run:
   ```
   # kubectl exec -it "$(for EACH in \
     $(kubectl get pods -n gpu-operator \
     -l app=nvidia-driver-daemonset \
     -o jsonpath={.items..metadata.name}); \
     do echo ${EACH}; done)" -n gpu-operator -- nvidia-smi
   ```

One should note, that most arguments to `helm install ...` above are
for the RKE2 variant of K8s. Some of them may be different for an 'upstream'
Kubernetes or may not be needed at all for it.
