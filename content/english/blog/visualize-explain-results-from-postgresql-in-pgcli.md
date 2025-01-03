---
title: "Visualize Explain Results from PostgreSQL in PGCLI"
meta_title: ""
description: "Recently I wrote a visualizer for explain query responses from postgresql. I also integrated it into pgcli..."
date: 2022-04-05T19:55:05Z
image: "https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2fzhn17xc70cnak0cxzo.png"
categories: ["Development", "Tools"]
author: "Daniel Kukula"
tags: ["postgresql", "pgcli", "tools"]
draft: false
---

Recently I wrote a [visualizer](https://github.com/dkuku/pyev) for explain query responses from PostgreSQL. I also integrated it into pgcli and it got merged few days ago. You can install it currently with:

```sh
pip install git+https://github.com/dbcli/pgcli@main
```

or wait for `3.4.2` release and update.
To switch between normal mode and explain mode you can press `F5`

![PGCLI Explain Visualizer](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2fzhn17xc70cnak0cxzo.png)
