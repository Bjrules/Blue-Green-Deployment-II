## Create Service Account, Role & Assign that role, And create a secret for Service Account and geenrate a Token

### Creating Service Account


```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

### Create Role 


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - secrets
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Bind the role to service account


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```
### Create Cluster role & bind to Service Account
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-cluster-role
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-cluster-role-binding
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: webapps
roleRef:
  kind: ClusterRole
  name: jenkins-cluster-role
  apiGroup: rbac.authorization.k8s.io

```
### Generate token using service account in the namespace

[Create Token](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=To%20create%20a%20non%2Dexpiring,with%20that%20generated%20token%20data.)

```
kubectl apply -f secret-file.yaml -n <namespace>
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sa-secret
  annotations:
    kubernetes.io/service-account.name: "jenkins"
type: kubernetes.io/service-account-token

```
#### kindly get the token using this command
```
kubectl describe secret sa-secret -n <namespace>

```
#### copy the token as it will be used for Jenkins-Kubernetes Authentication later.
```
[Kubernetes Secret for jenkins service account Token]
eyJhbGciOiJSUzI1NiIsImtpZCI6IjRYQzk0TVd2X21PTkhHeDVVMWE3Um5JbklCLUMwODNWaHFRMzVoeHhYcDQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ3ZWJhcHBzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InNhLXNlY3JldCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiOWVhMGMxNjktNTIyYS00NTNlLThmZmUtMjRiZGIzYWI1MDAwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OndlYmFwcHM6amVua2lucyJ9.a-CKgaUbmsPKb0mcK657kA7UkP8In7FdXoa-GfAksDfhc5ZUoAq-Eg-ckhkLQYG68Qw6TNfvFUiWgZ64B8xARzia1kAKDCDfXlZEv69GLc_MU5C7gtriuKePMxiJePCqUFQTYAFOb8X4d59bjEfsa5fNOcZIjAvmgj7VAcFyREP-n4nOdXgFhm6ulFGXA9v7JYmsVH2WFApH-NjXK8WNBKeS0Gi09nacw7xh9UxwiF2-IUVeXXrDxoegF5N5-uLLRP0DjQsmTPfRgqpnyih7yBdTp629t4EuavU3E1TG5Xsf44SvBTddk3Q613asQBRBSe-xgGVrmtmhhElGov4DwQ

```
