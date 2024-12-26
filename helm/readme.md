# manage multicluster deployment with helm

If your deployment are all managed by helm and argo you'll have to encapsulate the instruction [described here](../readme.md) with helm charts.

The primary helm chart make sure that the secondary cluster resource and primary rersources are removed in the right order with a pre-delete hooks.

## Do not bind the lifecycle of kasten deployment with the multicluster deployment

It's easy to create or use an [umbrella chart](https://helm.sh/docs/howto/charts_tips_and_tricks/#complex-charts-with-many-dependencies) that has a dependency to kasten and handle the multicluster deployment. But there is some caveats to this approach that I found difficult to solve : 
- the multicluster resource will be created before kasten is ready and the bootstraping of the primary cluster will fail 
- if you try to solve the above with a post-install hook then resources will be deleted and recreated at each upgrade which is really not what we want to do
- if I can separate features that can work idenpendently then it always proved to be a good choice to separate them. For instance if you have no secondary cluster to add for the moment, creating a primary cluster is cumbersome.

## install primary and secondary with helm 

here is an example you'll have to adapt the values files: 
on primary 
```
helm install k10-mc helm/k10-mc-primary -f helm/values/k10-mc-primary.yaml -n kasten-io-mc --create-namespace
```

on secondary
```
helm install k10-mc helm/k10-mc-secondary -f helm/values/k10-mc-secondary.yaml -n kasten-io
```

## install with argocd 

A similar action can be executed with argocd based on the helm chart

```
cat <<EOF | kubectl create -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: primary
  namespace: argocd
spec:
  destination:
    namespace: kasten-io-mc
    server: https://kubernetes.default.svc
  project: default
  sources:
  - chart: k10-mc
    repoURL: https://github.com/michaelcourcy/kasten-multicluster-argocd/helm/k10-mc-primary
    targetRevision: 0.1.0
    helm: 
      valueFiles:
      - $values/k10-mc-primary.yaml
  - repoUrl: https://github.com/michaelcourcy/kasten-multicluster-argocd/helm/values
    targetRevision: HEAD
    ref: values 
  syncPolicy:
    automated: 
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
cat <<EOF | kubectl create -f -
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: secondary
  namespace: argocd
spec:
  destination:
    namespace: kasten-io
    # adapt this value to your configuration
    server: https://kasten-se-guf1s564.hcp.westus.azmk8s.io:443
  project: default
  source:
    path: secondary
    repoURL: https://github.com/michaelcourcy/kasten-multicluster-argocd
    targetRevision: HEAD
  syncPolicy:
    automated: {}
EOF
```
