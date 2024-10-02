---
layout: post
title:  "Installing NVIDIA CUDA on Leap"
date:   2024-08-12
categories: nvidia cuda 'package groups'
author: Egbert Eich
---

NVIDIA offers multiple ways to install CUDA. For supported Linux distributions
- like openSUSE Leap - a particular convenient way to do so is to utilize
the distro's package manager and install packages he network. A network
installation is useful in particular when only a subset of the available
packages are needed - which we will address in this article.

# The Components of the CUDA Stack

Of course, the easiest way to install all of CUDA is to issue:
```
# zypper install cuda
```
This will get you all of the latest CUDA - including a number of other
components like for X and Wayland - but this is probably not what you
need. NVIDIA's instructions for a network install suggest to run
```
# zypper install cuda-toolkit-12-6
```
This, however, does not install the runtime components, like drivers,
so you won't be able to run applications on NVIDIA hardware.
