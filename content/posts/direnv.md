---
title: "Direnv"
featuredImg: "https://i.ibb.co/fqbKW5C/apprciate.jpg"
description: Appreciation post for direnv
tags: ["direnv", "shell-hacks", "tips-and-tricks"]
date: 2024-01-04T09:24:53+05:30
categories: ["productivity", "shell", "customisation"]
scrolltotop: true
---
I have recently been introduced to [direnv](https://direnv.net) by one of my friends. And boy-oh-boy am I grateful about finding this.

As an infrastructure engineer, my everyday task is to play around with things. Different Kubernetes cluster for example. And creating different commands or aliases to interact with different clusters becomes cumbersome sooner or later. Up until now, what I have been doing is to create my cluster (using [k3d](https://k3d.io/) or [kind](https://kind.sigs.k8s.io/)) and then create aliases to interact with these cluster changing `--kubeconfig` option with each. 

After configuring `.envrc` file, now I don't have to worry about remembering what command I had created.

Here is one sample. This is one of my directories' layout:
```shell
$ tree -a --charset terminal k3d/
k3d/
|-- dev
|   |-- cluster1.yaml
|   `-- .envrc
`-- prod
    |-- cluster2.yaml
    `-- .envrc

2 directories, 4 files
```

and content of `.envrc` file looks like this:
```shell
$ cat k3d/dev/.envrc
export KUBECONFIG="${HOME}/.config/k3d/kubeconfig-dev-cluster1.yaml"
```

Util next time... Peace :v:!
