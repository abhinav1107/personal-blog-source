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

{{< admonition info "Note" >}}
Everytime we make changes to this `.envrc` file, we need to approve it's content by running `direnv allow`.
```shell
$ cd ../prod/
direnv: error <rest-of-the-path>/k3d/prod/.envrc is blocked. Run `direnv allow` to approve its content
$ direnv allow
direnv: loading <rest-of-the-path>/k3d/prod/.envrc
direnv: export +KUBECONFIG
```
{{< /admonition >}}

{{< admonition type=tip title="Tip" >}}
I have added `.envrc` in my globally ignored list of git files, so that I don't end up pushing something in Git remote repo which should not be there, like credentials, tokens, etc. But YMMV.
{{< /admonition >}}

Util next time... Peace ✌🏻!
