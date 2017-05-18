# Azure Active Directory v2 Kubernetes OIDC Setup

## Requirements:
* Kubernetes 1.6+

## Get the helper app
* Make sure your PATH and GOPATH is configured correctly with `export PATH=$PATH:$GOPATH/bin`

* `go get github.com/sozercan/k8s-oidc-helper-azure` to grab the helper app

## Create an converged application

* Go to https://apps.dev.microsoft.com and create a converged app

* Create a web platform with `https://localhost` as redirect URL

* Create a password (`client_secret`)

* Create a file called `client_secret.json` with contents below:

```
{
  "installed": {
    "client_id": "<INSERT YOUR CLIENT ID>",
    "client_secret": "<INSERT YOUR CLIENT SECRET>"
  }
}
```

* Add `email`, `offline_access`, `openid`, `profile` and `User.read` as delegated Microsoft Graph permissions

* Run the helper with `./k8s-oidc-helper-azure -c client_secret.json`

* Login to app with your Microsoft account (MSA) or organizational account and accept the permission request

* Add generated user to your kubeconfig. Should look like

```
- name: <YOUR EMAIL>
  user:
    auth-provider:
      config:
        client-id: <YOUR CLIENT ID>
        client-secret: <YOUR CLIENT SECRET>
        id-token: <GENERATED ID TOKEN>
        idp-issuer-url: https://login.microsoftonline.com/<YOUR TENANT ID>/v2.0
        refresh-token: <GENERATED REFRESH TOKEN>
      name: oidc
```

* Create `clusterrolebinding` called `permissive-binding` for your `kubectl` client so you can create `clusterrolebindings` for users

```
kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=client \
  --group=system:serviceaccounts
```

* SSH in to your master node

`ssh -l <USER> <CLUSTERNAME>.<REGION>.cloudapp.azure.com`

* Edit `/etc/kubernetes/manifests/kube-apiserver.yaml` and add

```
        - "--authorization-mode=RBAC"
        - "--oidc-client-id=<INSERT YOUR CLIENT ID>"
        - "--oidc-issuer-url=https://sts.windows.net/<YOUR TENANT ID>/"
        - "--oidc-username-claim=sub"
```

* Go to https://jwt.io/ and paste your `id_token` and look for the field `sub`

* Create a file called `read_only.yaml` that defines the `clusterrolebinding` and make sure to edit with your tenant id and sub

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read_only
subjects:
  - kind: User
    name: 'https://sts.windows.net/<INSERT YOUR TENANT>/#<INSERT YOUR SUB>'
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

* Create the user with `kubectl create -f read_only.yaml`

* Since `view` `clusterrole` gives ability to `get pods`, this should work:
```
$ kubectl --user=<YOUR EMAIL> get pods
NAME                       READY     STATUS    RESTARTS   AGE
my-nginx-858393261-m687p   1/1       Running   0          7d
my-nginx-858393261-v5k9l   1/1       Running   0          6d
```

* However, we don't have permissions to create deployments, so `run` will fail:
```
$ kubectl run nginx --image=nginx --user=<YOUR EMAIL>
Error from server (Forbidden): User "https://sts.windows.net/<YOUR TENANT ID>/#<YOUR SUB>" cannot create deployments.extensions in the namespace "default". (post deployments.extensions)
```
