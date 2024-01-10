---
title: "Restarting Pods on ConfigMap or Secret Updates"
slug: "reloader"
summary: "How to use reloader to restart a pod automatically when it's ConfigMap or Secret changes"
date: 2023-12-23T17:51:07+05:30
tags: ["kubernetes", "reloader", "devops"]
---
In my current work setup, we have a lot of Kubernetes deployments. Most of them are Spring Boot applications, involving `application.properties` file. These properties files are loaded via ConfigMaps. Since the company is in early stages of application development, these config maps are changing frequently. Up until now, we have been performing manual rollout restart of deployments whose config has changed.

Pretty soon it became obviously that our manual process is not going to scale well, and we needed to do something about it. That's when we found this extremely wonderful project, [Reloader](https://github.com/stakater/Reloader).

Installing Reloader is rather very simple. There are multiple ways to do it. I went with helm chart way. Mostly because upgrading packages are lot easier using helm. This is what I did to install reloader in our EKS and GKE clusters:
- add helm repo:
  ```shell
  helm repo add stakater https://stakater.github.io/stakater-charts
  helm repo update
  ```
- create custom `values.yaml` file for the helm installation:
  ```shell
  cat <<EOF | tee ~/reloader-values.yaml
  reloader:
    autoReloadAll: true
  EOF
  ```
- install reloader:
  ```shell
  helm install reloader stakater/reloader -n reloader \
  --create-namespace -f ~/reloader-values.yaml
  ```

The `values.yaml` file can be found [here](https://github.com/stakater/Reloader/blob/master/deployments/kubernetes/chart/reloader/values.yaml). Most of the settings are self-explanatory. For us, the option `autoReloadAll` made more sense. This option basically starts reloader with argument `--auto-reload-all`.

Reloader, by default, will not watch for ConfigMaps/Secret changes, and we need to add extra annotation (like `reloader.stakater.com/auto`, etc.) to our deployments / daemonset / statefulset. But, when we use `--auto-reload-all` argument, all resources that do not have the auto annotation `reloader.stakater.com/auto: "false"`, will be reloaded automatically when their ConfigMaps or Secrets are updated.

This worked out so well for us that we sent it to production within 2 days of testing in development environments.

> I don't think necessity is the mother of invention. Invention, in my opinion, arises directly from idleness, possibly also from laziness - to save oneself trouble.
>
> \- Agatha Christie

Well, How about that??

Until next time!!

{{< figure src="/images/sloth.jpg" title="Me relaxing after saving time from manual restarts" >}}
