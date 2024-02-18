---
title: "ArgoCD App of Apps"
slug: argocd-app-of-apps
summary: "ArgoCD App of Apps using GitOps"
tags: ["argocd", "bootstraping", "gitops"]
categories: ["gitops", "argocd", "kubernetes"]
date: 2024-02-10T09:24:53+05:30
---
> Update:
>
> Since the time of writing this article, we came to know about ArgoCD ApplicationSet Controller and are trying that out.
> 
> It's under POC stage right now. I will update here when that's get implemented in production. (:

{{< alert "lightbulb" >}}
- [ApplicationSet Controller](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/) is the recommended way of app management.
- This article is not about ApplicationSet Controller. Documenting it here because it was a fun learning experience.
{{< /alert >}}

I was introduced to ArgoCD back in September 2022. Fell in love with it right away. Amazing tool (Checkout  [ArgoCD](https://argoproj.github.io/cd/) if you haven't). Initially, everytime a new service deployment came, I would go to the UI and add the service manually. This got out of hand really quickly. To handle this I wrote an Ansible playbook to do the addition for me. The playbook more or less looked like this:

```yaml
#
# About: This playbook will create app for ArgoCD
# Extra Variables:
#  required variables
#    argo_chart e.g. "nginx"
#
#  optional variables
#    argo_ns e.g. "apps"                          eks cluster namespace where app will be installed.
#    argo_app_name e.g. "nginx-tier0"             name that your would see in ArgoCD UI. if not given, name is calculated using chart name and tier
#    argo_tier e.g. "tier0"                       tier file for deployment. defaults to tier0.
#    repo_id e.g. "helmcharts"                    this will the key id in global vars file. defaults to helmcharts
#    cluster_url e.g. "kubernetes.default.svc"
#    revision_limit e.g. "5"                      defaults to 1
#    app_branch e.g. "main"                       branch name for the helm chart repo, defaults to main
#    sync_policy e.g. "none"                      defaults to automated. this is how the app syncing will be done. If automated, then synced automatically.
#    argo_project e.g. "default"                  default argocd project where apps will be created. defaults to default
---
- hosts: eks-controller-box
  become: true

  roles:
    - role: create-argocd-app
```

where the `create-argocd-app` ansible role would just run an extended version of `argocd` command and the app would appear in the UI.

I put this workflow behind Jenkins, reading the arguments and then passing them to Jenkins' exec statement. This is working and will continue to work without any issue for entirety of our setup. But somehow this didn't feel like a proper [GitOps](https://about.gitlab.com/topics/gitops/) way of getting things done. And this would bother me all the time. Finally, I decided to do something about it.

A quick google search to solve this problem took me to [ArgoCD Cluster Bootstrapping](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/). This article is about that.

The idea was pretty straight forward. I just had to create a helm chart, where all templates would have to be of `kind: Application`. I created a new Git repository, `argocd-apps-bootstrap`, and this was the directory structure:
```text
~/argocd-apps-bootstrap$ tree
.
├── README.md
├── apps
│   ├── infra
│   │   ├── Chart.yaml
│   │   ├── configs
│   │   │   └── dev.yaml
│   │   ├── templates
│   │   │   └── echoserver.yaml
│   │   └── values.yaml
│   └── platform
│       ├── Chart.yaml
│       ├── configs
│       │   ├── dev.yaml
│       │   ├── prod.yaml
│       │   └── staging.yaml
│       ├── templates
│       │   └── core.yaml
│       └── values.yaml
└── configs
    ├── common
    │   └── values.yaml
    ├── dev
    │   └── values.yaml
    ├── prod
    │   └── values.yaml
    └── staging
        └── values.yaml

12 directories, 15 files
~/argocd-apps-bootstrap$
```

Each file under repository's root `configs` directory would contain values for variables. For example, `./configs/common/values.yaml` would contain variables that would be common to all environments and all templates, so that I don't have to redeclare them in each file:
```text
~/argocd-apps-bootstrap$ cat ./configs/common/values.yaml
argocdNamespace: argocd
appNamesapce: apps
appProject: default
appServer: https://kubernetes.default.svc
appRevisionHistoryLimit: 2
appDefaultTier: tier0
~/argocd-apps-bootstrap$
```

and the files `./configs/<env-name>/values.yaml` would contain value of variables that would be specific to that environment.

then files like `./apps/infra/configs/dev.yaml` would contain template specific variables
```text
~/argocd-apps-bootstrap$ cat apps/others/configs/dev.yaml
echoserver:
  enabled: false
~/argocd-apps-bootstrap$
```

And then finally the template file itself will look something like this:
```text
{{- if .Values.echoserver.enabled }}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: echoserver-{{.Values.echoserver.tier | default .Values.appDefaultTier }}
  namespace: {{ .Values.argocdNamespace }}
  {{- with .Values.echoserver.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: {{ .Values.appNamesapce }}
    server: {{ .Values.appServer | squote }}

  project: {{ .Values.appProject }}
  revisionHistoryLimit: {{ .Values.appRevisionHistoryLimit }}

  source:
    helm:
      valueFiles:
        - ./environments/{{.Values.envName}}/{{.Values.echoserver.tier | default .Values.appDefaultTier }}/values.yaml
        - ./environments/{{.Values.envName}}/{{.Values.echoserver.tier | default .Values.appDefaultTier }}/images.yaml
    path: charts/echoserver
    repoURL: 'git@<some-git-repo-url-where-your-actual-helm-charts-are>/helm-charts.git'
    targetRevision: {{ .Values.echoserver.repoBranch }}

  syncPolicy:
    automated:
      selfHeal: true
      prune: true
{{- end }}
```

In this above block, the lines under `valueFiles` are referring to the files present under source of the helm chart repository `git@<some-git-repo-url-where-your-actual-helm-charts-are>/helm-charts.git` and not under this app bootstrap repository.

Once all these are done and commited to git repository, I simply added one app using ArgoCD ui, whose eventual manifest would look like this:
```yaml
project: default
source:
  repoURL: 'git@<git-repo-url>/argocd-apps-bootstrap.git'
  path: apps/infra
  targetRevision: main
  helm:
    valueFiles:
      - ../../configs/common/values.yaml
      - ../../configs/dev/values.yaml
      - ./configs/dev.yaml
destination:
  server: 'https://kubernetes.default.svc'
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```


That's it. From next time onwards, I just have to add files in this `argocd-apps-bootstrap` under their respective template folder. ArgoCD will sync the repo and do the rest for me. All I have to do now is add a file, raise a Merge Request and get it merged.

{{< alert >}}
**Warning!** Since this gives ability to add any applications to existing infra, ensure that the bootstrap repo itself is not writable without a review.
{{< /alert >}}


This whole setup has been working really well for us. Now, we don't need to run any Jenkins job. GitOps all the way.

Until next time. Peace :victory_hand: 
