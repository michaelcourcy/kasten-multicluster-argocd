# Goal

Often you want to use helm with argocd when you have multiple clusters to manage.

By separating template and variables you create a better normalization of your deployment accross your clusters.

# manage multicluster deployment with helm

We have encapsulated the instruction [described here](../readme.md) with helm charts.

The primary helm chart 
- is deployed in the kasten-io-mc namespace, this namespace can be created by helm or argocd.
- make sure that the secondary cluster resource and primary cluster rersources are removed in the right order with a pre-delete hooks.

The secondary helm chart is deployed in the alrady existing kasten-io namespace.


## Do not bind the lifecycle of kasten deployment with the multicluster deployment

It's easy to create or use an [umbrella chart](https://helm.sh/docs/howto/charts_tips_and_tricks/#complex-charts-with-many-dependencies) that has a dependency to kasten and handle the multicluster deployment. But there is some caveats to this approach that I found difficult to solve : 
- the multicluster resource will be created before kasten is ready and the bootstraping of the primary cluster will fail 
- if you try to solve the above with a post-install hook then resources will be deleted and recreated at each upgrade which is really not what we want to do
- if one can separate features that can work idependently then it's often proved to be a good choice to separate them. Making the code base easier to understand.

## install primary and secondary with helm 

here is an example you'll have to adapt the values files: 
on primary 
```
helm install k10-mc ./helm/k10-mc-primary -f helm/values/k10-mc-primary.yaml -n kasten-io-mc --create-namespace
```

on secondary
```
helm install k10-mc ./helm/k10-mc-secondary -f helm/values/k10-mc-secondary.yaml -n kasten-io
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
  - repoURL: https://github.com/michaelcourcy/kasten-multicluster-argocd
    path: helm/k10-mc-primary
    targetRevision: HEAD
    helm: 
      valueFiles:
      - \$values/helm/values/k10-mc-primary.yaml
  - repoURL: https://github.com/michaelcourcy/kasten-multicluster-argocd
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
  sources:
  - repoURL: https://github.com/michaelcourcy/kasten-multicluster-argocd
    path: helm/k10-mc-secondary
    targetRevision: HEAD
    helm: 
      valueFiles:
      - \$values/helm/values/k10-mc-secondary.yaml
  - repoURL: https://github.com/michaelcourcy/kasten-multicluster-argocd
    targetRevision: HEAD    
    ref: values
  syncPolicy:
    automated: 
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```
