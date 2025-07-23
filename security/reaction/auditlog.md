# Setup audit logging 

## Hints (this is on a system, where kubernetes-api-server runs as static pod, e.g. kubeadm)

  * When the config unter /etc/kubernetes/manifests/kupe-apiserver.yaml changes
    * kubelet automatically detects this and restarts the server
    * if there is a misconfig the pod will vanish

```
# There are 4 stages, that can be monitored:
```
![image](https://github.com/user-attachments/assets/39a132aa-0f26-457a-9f2c-4dc3a55f4ee5)


## Step 1: 1st -> session (on control plane): Watch kube-apiserver - pod on controlplane 

```
# we want to 
kubectl -n kube-system get pods -w 
```

## Step 2: 2nd -> session (on control plane): create a policy

```
cd /etc/kubernetes
```

```
nano audit-policy.yaml
```

```
apiVersion: audit.k8s.io/v1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"

```

```
# Important: You do not need to apply/create it.
```

## Step 3: 2nd -> session: Change settings in /etc/kubernetes/manifests/kube-apiserver.yaml 

```
# security copy
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root
```

```
# Add lines in /etc/kubernetes/manifests/kube.apiserver.yaml 
- --audit-log-path=/var/log/kubernetes/apiserver/audit/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policies.yaml
```

```
# Add lines under volumeMounts
- mountPath: /etc/kubernetes/audit-policy.yaml
  name: audit
  readOnly: true
- mountPath: /var/log/kubernetes/apiserver/audit/
  name: audit-log
  readOnly: false
```

```
# Add volumes lines under volumes
- name: audit 
  hostPath: 
    path: /etc/kubernetes/audit-policy.yaml 
    type: File 

- name: audit-log 
  hostPath: 
    path: /var/log/kubernetes/audit/ 
    type: DirectoryOrCreate 

```

## Step 4: 1st -> session 

![image](https://github.com/user-attachments/assets/3a286912-d179-4de4-bbb6-513f9e611259)c

## Step 5: 2nd -> session

```
# There should be enought noise already
cat /var/log/kubernetes/audit/audit.log
```

## Reference 

 * https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/



