---
title: "Machine Learning on Ryzen 7840HS"
meta_title: ""
description: "I bought an amd laptop with 7840hs and 32GB ram with the hope that I can play with machine learning models..."
date: 2024-02-15T19:50:19Z
image: "/images/image-placeholder.png"
categories: ["Development", "Machine Learning"]
author: "Daniel Kukula"
tags: ["machine-learning", "amd", "ryzen", "pytorch"]
draft: false
---

I bought an AMD laptop with 7840hs and 32GB RAM with the hope that I can play with machine learning models. At the beginning it was problematic, I could not get anything bigger to work but after some time spent reading and experimenting here are some of my findings.

I'm using Linux and I have the rocm packages installed.
I have these env variables exported:

```
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export PYTORCH_ROCM_ARCH="gfx1100"
```

The integrated graphics on this processor has a unified memory architecture. It means that the memory is shared between CPU and GPU.

The GPU memory can be set in BIOS under the UMA size option.
But this is the minimal memory amount. When a game needs more memory it can request more memory. Can a model take advantage of that? It turns out that it can, at least it's implemented in llama.cpp, but it's not enabled by default. I managed to load mixtral quantized to 4bits fully loaded into memory and run interference on that. You can search the [readme for hipblas](https://github.com/ggerganov/llama.cpp#hipblas).

> it is also possible to use unified memory architecture (UMA) to share main memory between the CPU and integrated GPU by setting -DLLAMA_HIP_UMA=ON". However, this hurts performance for non-integrated GPUs (but enables working with integrated GPUs).

But what with other frameworks?
Most of the frameworks support rocm somehow. For example pytorch has dedicated [builds that you install with this command](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/how-to/3rd-party/pytorch-install.html):

```
pip3 uninstall torch torchvision torchaudio
pip3 install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/rocm5.6/
```

The problem here is that pytorch does not support the shared memory yet and we are limited by the maximum memory that you can set in BIOS. My BIOS only allows to set 4GB of total 32GB my machine has.

I found a workaround to this by using [Smokeless_UMAF](https://github.com/DavidS95/Smokeless_UMAF) that allows to access settings that may not be exposed in normal BIOS.
Thanks to it I could set 16GB of RAM to iGPU.

Now when I load comfyUI I'm greeted with this text:

```
Total VRAM 16384 MB, total RAM 15677 MB                                         
Set vram state to: NORMAL_VRAM                                                  
Device: cuda:0 AMD Radeon Graphics : native  
```

This may be risky. There is a known problems section in the readme. My BIOS loads but the page where I can set the size of shared memory hangs so there is also a problem with my laptop.
On the other hand I don't change it very often so I can live with that.
