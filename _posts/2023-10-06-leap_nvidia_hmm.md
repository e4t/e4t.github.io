---
layout: post
title:  "Simplify GPU Application Development with HMM on Leap"
date:   023-10-06
categories: nvidia, hmm, cuda
---

Authors: Christian Goll, Egbert Eich

Recently, NVIDIA has introduced Heterogeneous Memory Management (HMM)
in its [open source kernel drivers](https://github.com/NVIDIA/open-gpu-kernel-modules/)
which [simplifies GPU Application Development](https://developer.nvidia.com/blog/simplifying-gpu-application-development-with-heterogeneous-memory-management/).
It unifies system memory access across CPUs and GPUs and removes the
need to copy memory content between CPU and GPU memory.
It extends Unified Memory to cover both system allocated memory as well
as memory allocated by `cudaMallocManaged()`.

You may ask, "how do I make this work on my Leap system?"
If you are a Leap 15.5 user, the open driver is already available to you.
Therefore, if you have an NVIDIA chipset with a GPU System Processor (GSP),
ie. Turing or later, we have you covered. Here is how:

## Installation on openSUSE Leap 15.5

The simplest way to accomplish this is to login as root and run
the following commands in your shell:
```
zypper ar https://developer.download.nvidia.com/compute/cuda/repos/opensuse15/x86_64/cuda-opensuse15.repo
zypper --gpg-auto-import-keys refresh
zypper -n install -y --auto-agree-with-licenses --no-recommends nvidia-open-gfxG05-kmp-default cuda
```
This will add the NVIDIA CUDA repository and install CUDA with the kernel
modules required.  
Do you require signed drivers to support secure boot or deploy in a public
cloud environment? In this case, instead of the above, execute:
```
zypper ar https://developer.download.nvidia.com/compute/cuda/repos/opensuse15/x86_64/cuda-opensuse15.repo
zypper ar https://download.nvidia.com/opensuse/leap/15.5/ NVIDIA-drivers
zypper --gpg-auto-import-keys refresh
zypper -n in -y --auto-agree-with-licenses --no-recommends nvidia-open-driver-G06-signed-kmp-default nvidia-drivers-minimal-G06 cuda
```
This makes use of the NVIDIA open driver package shipped and signed by
SUSE - like the rest of your kernel. This eliminates the need to enroll
a MOK as well an extra build stage when the kernel drivers are installed
or updated. Thus it helps to reduce the size of the cloud image by
removing the need for extra build tools.  
To use these kernel drivers, it installs a set of user space driver
packages which are not yet available in the CUDA software repository.

## Preparations

For chipsets with a display engine (i.e. which have display outputs), the open driver
support is still considered alpha. Therefore, you may have to add or uncomment
the following option in /etc/modprobe.d/50-nvidia-default.conf:
```
options nvidia NVreg_OpenRmEnableUnsupportedGpus=1
```
Once these steps have been performed, you may either reboot the system or run
```
modprobe nvidia
```
as root to load all required kernel modules.

## Testing the Installation

To check if HMM is available and enabled, query the 'Addressing Mode' property:
```
nvidia-smi -q | grep Addressing
Addressing Mode : HMM
```
If you see above output, HMM is available on your system.

## Compile HMM Sample Code

NVIDIA discusses some code examples for HMM in its [blog
post](https://developer.nvidia.com/blog/simplifying-gpu-application-development-with-heterogeneous-memory-management/).
The examples are stored in [github](https://github.com/NVIDIA/HMM_sample_code). If you
would like to try out the examples, here are some hints on building and running them.  
Some these need a newer gcc than the stock version shipped with Leap 15,
which you can install with:
```
zypper in gcc12-c++
```

In order to compile the examples, the `PATH` environment variable needs to be extended to
point to the CUDA binaries:

```
export PATH=/usr/local/cuda/bin/:${PATH}
```

You may now compile the examples under the path `src` using the following commands:
```
nvcc -std=c++20 -ccbin=/usr/bin/g++-12 atomic_flag.cpp -o atomic_flag
nvcc -std=c++20 -ccbin=/usr/bin/g++-12 file_after.cpp -o file_after
nvcc -std=c++20 -ccbin=/usr/bin/g++-12 file_before.cpp -o file_before
nvcc -std=c++20 -ccbin=/usr/bin/g++-12 ticket_lock.cpp -o ticket_lock
```

### 'weather_app' Example

For this example application, the system gcc compiler is sufficient. Only `$PATH` has to be
set to
```
export PATH=/usr/local/cuda/bin/:${PATH}
```
Now, build the binary `weather_app` by running
```
make
```

The [blog by NVIDIA](https://developer.nvidia.com/blog/simplifying-gpu-application-development-with-heterogeneous-memory-management/)
describes how to obtain the data required to run the app.
If you're unable to download the ~1.3 TB of data, you may also use the random data generator from
this [PR on github](https://github.com/NVIDIA/HMM_sample_code/pull/3).
The random data app can be compiled with
```
g++ create_random_data.cpp -o create_random_data -O2 -Wall
```
The application has no command line parameters, and the start and end year for the random data
has to be set in the source code itself.

---
**NOTE**
If your graphic card doesn't have sufficient VRAM to run the original sample code, you may
scale down the data size by reducing the `input_grid_height` and `input_grid_width` parameters
in both `create_random_data.cpp` and `weather_app.cu`.

---

To do a sample run:
```
mkdir binary_1hr_all
./weather_app
./weather_app 1981 1982 binary_1hr_all/
```

---
**NOTE**
The `Makefile` doesn't compile CUDA kernels for the Turing GPUs and also has a faulty error message handling. You might want check out [https://github.com/NVIDIA/HMM_sample_code/pull/2](https://github.com/NVIDIA/HMM_sample_code/pull/2) which fixes this issues.

---

# Summary

- The NVIDIA open driver provides HMM ([Heterogeneous Memory
Management](https://developer.nvidia.com/blog/simplifying-gpu-application-development-with-heterogeneous-memory-management/))
  which extends the simplicity of the CUDA Unified Memory programming
  model even further on supported chipsets [^1] by including
  system allocated memory.
- HMM is available for openSUSE Leap 15.5.
- The open driver allows for pre-built kernel drivers signed by SUSE.
  - This greatly simplifies the installation in a secure boot environment.
  - It streamlines the installation in public cloud environments by
	eliminating an extra build stage and reducing the size of the final
	image. 
- We have demonstrated how to install and test HMM on Leap 15.5.

---
[^1]: Turing and later
