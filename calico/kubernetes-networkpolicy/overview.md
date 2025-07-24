# Overview (some basics) 

## Version 3.30 

  * Introducted Tiers

## Ordering  

  * For the default deny GlobalNetworkPolicy use no order
    * it will then be evaluated as last rule (catch all)

  * **Ref:** https://docs.tigera.io/calico-cloud/network-policy/default-deny

## Implicit Deny 

  * the selector() defines whom the NetworkPolicy or GlobalNetworkPolicy are for
  * If no rules apply -> it will be an implicit deny



## crd.projectcalico.org/v1 vs projectcalico.org/v3

```
Long story short: Please only use projectcalico.org/v3
They also work with kubectl (when calico apiserver is running - this is the case by default)
kubectl -n calico-system get pods | grep api
```

```
LONG STORY:
Don't touch crd.projectcalico.org/v1 resources. They are not currently supported for end-users and the entire API group is only used internally within Calico. Using any API within that group means you will bypass API validation and defaulting, which is bad and can result in symptoms like # 2 above. You should use projectcalico.org/v3 instead. Note that projectcalico.org/v3 requires that you install the Calico API server in your cluster, and will result in errors similar to # 1 above if the Calico API server is not running
```
  * Ref: https://github.com/projectcalico/calico/issues/6412
