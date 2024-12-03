## Annex 1. Kasten is installed on both instances
As a prequisite we suppose that Kasten has been installed on both primary and secondaries with proper ingress using valid https certificate. Here is an [end to end guide](https://veeamkasten.dev/kasten-on-eks-2) for installing an ingress controller with certmanager to expose Kasten dashboard. 

> **Note on installing the ngninx ingress controller on azure** 
> For  installing the ngninx controller on AKS you'll have to follow this [guide](https://learn.microsoft.com/en-us/azure/linux-workloads/createaksdeployment/readme) instead of the operations described on the previous [link](https://veeamkasten.dev/kasten-on-eks-2).

In our case Kasten is exposed in 
- https://mcourcy-primary.customers.kastenevents.com/k10/
- https://mcourcy-secondary.customers.kastenevents.com/k10/
