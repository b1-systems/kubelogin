# Getting Started with Keycloak

## Prerequisite

- You have an administrator role of the Keycloak realm.
- You have an administrator role of the Kubernetes cluster.
- You can configure the Kubernetes API server.
- `kubectl` and `kubelogin` are installed.

## 1. Setup Keycloak

Open the Keycloak and create an OIDC client as follows:

- Client ID: `kubernetes`
- Valid Redirect URL: `http://localhost:8000/`
- Issuer URL: `https://keycloak.example.com/auth/realms/YOUR_REALM`

You can associate client roles by adding the following mapper:

- Name: `groups`
- Mapper Type: `User Client Role`
- Client ID: `kubernetes`
- Client Role prefix: `kubernetes:`
- Token Claim Name: `groups`
- Add to ID token: on

For example, if you have the `admin` role of the client, you will get a JWT with the claim `{"groups": ["kubernetes:admin"]}`.

## 2. Setup Kubernetes API server

Configure your Kubernetes API server accepts [OpenID Connect Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens).

If you are using [kops](https://github.com/kubernetes/kops), run `kops edit cluster` and add the following spec:

```yaml
spec:
  kubeAPIServer:
    oidcIssuerURL: https://keycloak.example.com/auth/realms/YOUR_REALM
    oidcClientID: kubernetes
    oidcGroupsClaim: groups
```

## 3. Setup Kubernetes cluster

Here assign the `cluster-admin` role to the `kubernetes:admin` group.

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: keycloak-admin-group
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: Group
  name: kubernetes:admin
```

You can create a custom role and assign it as well.

## 4. Setup kubectl

Configure `kubectl` for the OIDC authentication.

```sh
kubectl config set-credentials NAME \
  --auth-provider oidc \
  --auth-provider-arg idp-issuer-url=https://keycloak.example.com/auth/realms/YOUR_REALM \
  --auth-provider-arg client-id=kubernetes \
  --auth-provider-arg client-secret=YOUR_CLIENT_SECRET
```

## 5. Run kubelogin

Run `kubelogin`.

```
% kubelogin
2018/08/10 10:36:38 Reading .kubeconfig
2018/08/10 10:36:38 Using current context: hello.k8s.local
2018/08/10 10:36:41 Open http://localhost:8000 for authorization
2018/08/10 10:36:45 GET /
2018/08/10 10:37:07 GET /?state=...&session_state=...&code=ey...
2018/08/10 10:37:08 Updated .kubeconfig
```

Now your `~/.kube/config` should be like:

```yaml
users:
- name: hello.k8s.local
  user:
    auth-provider:
      config:
        idp-issuer-url: https://keycloak.example.com/auth/realms/YOUR_REALM
        client-id: kubernetes
        client-secret: YOUR_SECRET
        id-token: ey...       # kubelogin will update ID token here
        refresh-token: ey...  # kubelogin will update refresh token here
      name: oidc
```

Make sure you can access to the Kubernetes cluster.

```
% kubectl get nodes
NAME                                    STATUS    ROLES     AGE       VERSION
ip-1-2-3-4.us-west-2.compute.internal   Ready     node      21d       v1.9.6
ip-1-2-3-5.us-west-2.compute.internal   Ready     node      20d       v1.9.6
```