---
title: "Livebook on Steam Deck"
meta_title: ""
description: "This post may be updated in the future. Steam deck is a modern machine. I wanted to test how long it takes to generate images..."
date: 2023-01-19T20:23:25Z
image: "/images/image-placeholder.png"
categories: ["Development", "Tutorial"]
author: "Daniel Kukula"
tags: ["elixir", "livebook", "steam-deck", "docker"]
draft: false
---

This post may be updated in the future.

Steam deck is a modern machine. I wanted to test how long it takes to generate images using stable diffusion on it.
Thanks to [livebook](https://livebook.dev) this becomes trivial. The biggest problem is to install the package. I did it using ssh but it can also be done directly on deck.

You need first switch to desktop mode.
From here you need to open Konsole and setup password for your user:
```sh
passwd
```

Next thing is to enable sshd server:
```sh
sudo systemctl start sshd
```

Having ssh enabled you can use your laptop to connect:
```sh
ssh deck@steamdeck
```

Next you need to install docker:
```sh
# enables the root partition to be rw
sudo steamos-readonly disable
# syncs the system
sudo pacman -Syyu
# updates stored signing keys
sudo pacman-key --refresh-keys
sudo pacman -S archlinux-keyring
# syncs the system again with new keys
sudo pacman -Syu
# installs docker
sudo pacman -S docker
# optional will start docker every time the deck starts
sudo systemctl enable docker
# starts docker daemon for current session
sudo systemctl start docker
```

Having docker installed we can run any docker image - so lets process with docker - it's basically the same command as from the livebook read me - just the port is changed, the suggested one is already in use.
```sh
sudo docker run -p 8888:8080 -p 8081:8081 --pull always -u $(id -u):$(id -g) -v $(pwd):/data livebook/livebook
```

After running this command you'll get an url printed in the terminal. I connected to it using my browser but I had to change the host from `http://0.0.0.0:8080` to `http://192.168.1.123:8888` - your steam deck ip may be different - I had to get mine from my router settings.

Next you can follow the official video from Jose:

[Announcing Bumblebee, GPT2, Stable Diffusion and more in Elixir](https://news.livebook.dev/announcing-bumblebee-gpt2-stable-diffusion-and-more-in-elixir-3Op73O)

The initial model download takes 5-20min depending on your internet connection. Stable diffusion runs on the steam deck but it's not very performant because it's running on the CPU - image generation takes around 8 minutes.
You can try other models also.
