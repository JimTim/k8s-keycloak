# Keycloak (Kustomize)

The deployment files are created with the help of [Kustomize](https://kustomize.io/). This allows image versions, secrets or config maps to be exchanged quickly. In addition, there is a small form of templating.

## Deployment

For the Kustomize deployment, this repo should only serve as a basis. All customisations should happen in an [overlay folder](https://github.com/kubernetes-sigs/kustomize/blob/master/README.md#2-create-variants-using-overlays). This includes the encrypted generation of a secret with the help of sops, the adjustment of the environment variables in the configmap and the exchange of the image version.

### Patches

Some patches for the [k3s](https://k3s.io/) installation

#### ingress.yaml

```yaml
## replace ingress class
### nginx
#- op: add
#  path: "/metadata/annotations/kubernetes.io~1ingress.class"
#  value: nginx
### traefik
- op: add
  path: "/metadata/annotations/kubernetes.io~1ingress.class"
  value: traefik

## replace host
- op: replace
  path: /spec/rules/0/host
  value: ghost.example.com
- op: add
  path: /spec/tls
  value: 
    - hosts: 
        - ghost.example.com

## cert-manager
- op: add
  path: /spec/tls
  value: 
    - hosts: 
        - ghost.example.com
      secretName: 'nerdware-de-tls' 
- op: add
  path: "/metadata/annotations/cert-manager.io~1cluster-issuer"
  value: letsencrypt-prod


## additional ingress patches
### nginx ingress patches
#- op: add
#  path: "/metadata/annotations/nginx.ingress.kubernetes.io~1ssl-redirect"
#  value: "false"
#- op: add
#  path: "/metadata/annotations/nginx.ingress.kubernetes.io~1backend-protocol"
#  value: "HTTP"
#- op: add
#  path: "/metadata/annotations/nginx.ingress.kubernetes.io~1proxy-body-size"
#  value: 300m
#- op: add
#  path: "/metadata/annotations/nginx.ingress.kubernetes.io~1proxy-connect-timeout"
#  value: 300
#- op: add
#  path: "/metadata/annotations/nginx.ingress.kubernetes.io~1proxy-read-timeout"
#  value: 300
#- op: add
#  path: "/metadata/annotations/nginx.ingress.kubernetes.io~1proxy-send-timeout"
#  value: 300
```

#### nodeselector.yaml

```yaml
- op: add
  path: "/spec/template/spec/nodeSelector"
  value: 
    kubernetes.io/hostname: "k8s-worker"
        
```

#### storageclass.yaml

```yaml
- op: add
  path: "/spec/storageClassName"
  value: "local-path"
```

### Secrets

Since a secret is already predefined in this base, a secret in the overlay should look like this.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: keycloak
  labels:
    app.kubernetes.io/name: keycloak
    app.kubernetes.io/service: auth
    app.kubernetes.io/component: app
  annotations:
    kustomize.config.k8s.io/needs-hash: "true"
    kustomize.config.k8s.io/behavior: replace
type: Opaque
stringData:
  admin-username: admin
  admin-password: <supersecretpassword>
  management-username: management
  management-password: <supersecretpassword>
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres
  labels:
    app.kubernetes.io/name: postgres
    app.kubernetes.io/service: auth
    app.kubernetes.io/component: database
  annotations:
    kustomize.config.k8s.io/needs-hash: "true"
    kustomize.config.k8s.io/behavior: replace
type: Opaque
stringData:
  username: root
  password: <supersecretpassword>
  database: keycloak

```

- **kustomize.config.k8s.io/needs-hash** --> A hash is generated on the secret file. This restarts the deployment if the secret changes.
- **kustomize.config.k8s.io/behavior** --> This annotation is essential, otherwise the existing secret from this base cannot be overwritten.

### Command

```shell
kustomize build --enable-alpha-plugins overlay/instance | less
kustomize build --enable-alpha-plugins overlay/instance | kubectl diff -f -
# and after all checks
kustomize build --enable-alpha-plugins overlay/instance | kubectl apply -f -
```
