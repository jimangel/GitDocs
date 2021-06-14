# GitDocs
A GitOps approach for hosting self-updating docs on Kubernetes - similar to Netlify.

With GitDocs you can create markdown documents in a central git repository and easily access them on a Kubernetes cluster. The git sync will occur every 30 seconds and can be set with the `GIT_SYNC_WAIT` environment variable in `examples/base/gitdocs-deployment.yaml`.

GitDocs is not an application, it is the "glue" between 3 microservices ([git-sync](https://github.com/kubernetes/git-sync), [hugo](https://gohugo.io), [nginx](https://www.nginx.com)) that gains superpowers when ran in a pod on Kubernetes.

![](https://i.imgur.com/VIe32Ai.png)


By containing the entire documentation lifecycle a pod, all you need is:
- access to a git repo
- access to a kubernetes cluster (v1.14 or later)

GitDocs is designed to run anywhere, including on-prem, and supports private or public git repositories.

## Quick start

1) Clone repo

    ```
    git clone https://github.com/jimangel/GitDocs
    cd GitDocs
    ```

1) Apply the configmap and deployment

    The nginx.conf is created as a separate configmap to allow you to tweak any settings.

    ```
    kubectl apply -f examples/base/nginx-configmap.yaml
    kubectl apply -f examples/base/gitdocs-deployment.yaml
    ```

1) Use kubectl proxy to preview

    ```
    kubectl port-forward deployment/gitdocs 8080:8080
    ```

    Open a browser to http://localhost:8080 and browse.

## Clean up

```
kubectl delete -f examples/base/nginx-configmap.yaml
kubectl delete -f examples/base/gitdocs-deployment.yaml
```

## Deploy with Helm

```
helm repo add jimangel https://jimangel-charts.storage.googleapis.com
helm repo update

helm install gitdocs jimangel/gitdocs
# https://github.com/jimangel/GitDocs/blob/master/helm-chart/values.yaml

# use a custom website repo
helm install gitdocs jimangel/gitdocs --set gitSync.env.gitSyncRepo="https://github.com/kubernetes/website.git"

# add an ingress object
helm install gitdocs jimangel/gitdocs --set gitSync.env.gitSyncRepo="https://github.com/kubernetes/website.git" --set ingress.enabeld="true" --set ingress.hosts[0]="mysite.example.com"

# TODO: some sites are not formatted the same, the "arg" should be more dynamic to account for that.
# TODO: clean up all kustomize with helm alternatives.
```

## Going further

From here you can create your own ingress and TLS strategy. Also, by abstracting the nginx config, you could always provide the SSL configuration to the pod.

Expose pod with a service:

```
kubectl expose deployment/gitdocs --port 8080
```

Sample ingress:

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: gitdocs
spec:
  rules:
    - host: gitdocs.example.com
      http:
        paths:
          - backend:
              serviceName: gitdocs
              servicePort: 8080
            path: /
#  tls:
#  - hosts:
#    - gitdocs.example.com
#    secretName: gitdocs-cert
EOF
```

## Deploy  with kustomize

Kustomize is a manifest generator that is build in to `kubectl` since version 1.14. If you're not familiar with Kustomize, please understand the following:

- `example/base` is where the entire deployment lives. You can take those files out and hand modify to deploy if you wanted (please don't).
- `example/overlays/...` is where any CHANGES live. This way, kustomize will merge the changes with the core documents before deploying.
- You can have as many different `overlays` as you'd like while sharing the same `base`

If you have internet access you can create directly with a url (no need to clone) for example:

```
kubectl apply -k github.com/jimangel/GitDocs//examples/overlays/kubernetes-website
```

**Note:** Depending on the size of the site it might take awhile to for the git clone. Check hugo [logs](#troubleshooting) to know when built.

### Deploy the official [Kubernetes docs](https://github.com/kubernetes/website)

```
kubectl apply -k examples/overlays/kubernetes-website
kubectl port-forward deployment/gitdocs 8080:8080
kubectl delete -k examples/overlays/kubernetes-website
```

### Deploy the official [KinD docs](https://github.com/kubernetes-sigs/kind)

```
kubectl apply -k examples/overlays/kind-website
kubectl port-forward deployment/gitdocs 8080:8080
kubectl delete -k examples/overlays/kind-website
```

### Deploy the [docsy](https://github.com/google/docsy/) example site

```
kubectl apply -k examples/overlays/docsy
kubectl port-forward deployment/gitdocs 8080:8080
kubectl delete -k examples/overlays/docsy
```

### Deploy a private repo via SSH

1) Generate an SSH keypair
    ```
    ssh-keygen -f /home/<USERNAME>/git-docs-ssh
    ```

1) Add public key to git repo (from the GUI)

    This setting is usually in **Repository settings** > **Deploy** or **Access** keys)

    ```
    cat /home/<USERNAME>/git-docs-ssh.pub
    ```

1) Test SSH access to repository manually first

    ```
    # add to your SSH keys
    ssh-add ~/git-docs-ssh
    
    # clone repo using SSH (should NOT prompt for password)
    git clone git@<GIT REPO>:<REPO PATH>.git
    ```

1) Create SSH key as Kubernetes secret

    ```
    kubectl create secret generic git-docs-ssh-key --from-file=ssh=/home/<USERNAME>/git-docs-ssh
    ```

1) Edit REPO URL in `patch.yaml` deployment patch.

    ```
    vi examples/overlays/private-repo/static-git-url.yaml   
    
    #env:
    #- name: GIT_SYNC_REPO
    #  value: "git@<GIT REPO>:<REPO PATH>.git"
    ```

1) Deploy private-repo overlay

    ```
    kubectl apply -k examples/overlays/private-repo
    kubectl port-forward deployment/gitdocs 8080:8080
    kubectl delete -k examples/overlays/private-repo
    ```

## Troubleshooting

```
NAMESPACE EVENTS: kubectl get events -n <NAMESPACE>

# set POD variable for next commands
# helm: POD=$(kubectl get pod -l app.kubernetes.io/name=gitdocs -o jsonpath="{.items[0].metadata.name}")
POD=$(kubectl get pod -l name=gitdocs -o jsonpath="{.items[0].metadata.name}")

# git-sync
LOGS: kubectl logs -f $POD -c git-sync
EXEC: kubectl exec -it $POD -c git-sync -- /bin/sh
DOCKER: docker run --rm -it \
  -v /tmp/git-data:/tmp/git --entrypoint /bin/sh \
  k8s.gcr.io/git-sync:v3.1.5

# hugo
LOGS: kubectl logs -f $POD -c hugo
EXEC: kubectl exec -it $POD -c hugo -- /bin/sh
DOCKER: docker run --rm -it \
  -v /tmp/git-data:/tmp/git --entrypoint /bin/sh \
  klakegg/hugo:0.65.2-ext-alpine

# nginx
LOGS: kubectl logs -f $POD -c nginx
EXEC: kubectl exec -it $POD -c nginx -- /bin/sh
CONFIG: kubectl edit cm nginx-conf -o yaml
```

## Change version of hugo

If using kustomize (see [example overlays](examples/overlays/)) change the  `HUGO_VERISON` environment variable in `patch.yaml`

Alternately change the hugo image tag in `examples/base/gitdocs-deployment.yaml`. For example: klakegg/hugo:**0.65.2**-ext-alpine. More info: https://github.com/klakegg/docker-hugo. **This might cause issues with permissions if using an image <0.70.0.**

## Running offline / air-gapped

There's two ways to run the klakegg/hugo container, with an edge tag or with a version tag:

- edge tag: `klakegg/hugo:edge-ext-alpine`
  - uses the latest container built by klakegg
  - the version is specified as a `HUGO_VERISON` environment variable
  - **the `HUGO_VERISON` is downloaded from the internet when container starts**
- version tag: `klakegg/hugo:0.65.2-ext-alpine`
  - uses the specific version in the tag and does not download from the internet
  - no need to set `HUGO_VERISON` environment variable

The reason GitDocs uses the edge build is because there was a issue running the container with a non-root user. This was [fixed](https://github.com/klakegg/docker-hugo/issues/24) in klakegg/hugo releases `>=0.70.1`.

To run in an air-gapped environment, you can remove the `HUGO_VERISON` environment variable and either use klakegg/hugo releases `>=0.70.1` (ex: `klakegg/hugo:0.70.1-ext-alpine`) or add `runAsUser: 0` in the hugo container's `securityContext:` and use an older tagged version (ex: `klakegg/hugo:0.65.2-ext-alpine`).

There still might be issues running in air-gapped environments if your website content calls any external resources.

## TODO:

- Security
  - Create network policies to isolate this pod
  - More hardening / vuln scanning
- Makefile
  - for deploying, building, and updating

## Thanks

A lot of this is built on top of the example present in the git-sync repo: https://github.com/kubernetes/git-sync/tree/master/demo
