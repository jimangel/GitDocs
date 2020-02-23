# GitDocs
A GitOps approach for hosting self-updating docs on Kubernetes - similar to Netlify.

With GitDocs you can create markdown documents in a central git repository and easily access them on a Kubernetes cluster. The documents will self-update as changes are detected in the git repo.

GitDocs is not an application, it is the "glue" between 3 microservices ([git-sync](https://github.com/kubernetes/git-sync), [hugo](https://gohugo.io), [nginx](https://www.nginx.com)) that gains superpowers when ran in a pod on Kubernetes.

![](https://i.imgur.com/VIe32Ai.png)


By containing the entire documentation lifecycle a pod, all you need is:
- access to a git repo
- access to a kubernetes cluster (v1.14 or later)

GitDocs is designed to run anywhere, including on-prem, and supports private or public git repositories.

## Quick start

1) Clone repo
    ```
    git clone https://github.com/jimangel/gitdocs
    cd gitdocs
    ```
1) Apply the configmap and deployment

    The nginx.conf is created as a separate configmap to allow you to tweak any settings.

    ```
    kubectl apply -f examples/base/nginx-configmap.yaml
    kubectl apply -f examples/base/gitdocs-deployment.yaml
    ```

1) Use kubectl proxy to preview
    ```
    kubectl port-forward deployment/blog 80:8080
    ```

    Open a browser to http://localhost and browse.

## Going further

Ingress strategies have been purposely excluded from getting started. From here you can create your own ingress and TLS strategy. Also, by abstracting the nginx config, you could always provide the SSL configuration to the pod.

## TODOs:

Priority #1: Security
- Create network policies to isolate this pod
- See what limitations are present building images from non-root & scratch
- More hardening / vuln scanning

Priority #2: Makefile
- for deploying, building, and updating

Other:
- create a template for docs !
- more examples (k8s.io, jimangel.io, etc)
- improve all the content after this

## Apply with kustomize (-k) - making it your own

(coming soon)

Kustomize is a manifest generator that is build in to `kubectl` since version 1.14. If you're not familiar with Kustomize, please understand the following:

- `example/base` is where the entire deployment lives. You can take those files out and hand modify to deploy if you wanted (please don't).
- `example/overlays/hello-world` is where any CHANGES live. This way, kustomize will merge my changes with the core documents before deploying.
- You can have as many different `overlays` as you'd like while sharing the same `base`

Taking all of the defaults, we will build an example site using a template I built. 

```
kubectl apply -k examples/overlays/hello-world/
```

I will add additional examples covering using your own images or setting up git-sync with an SSH key.

## Private repos

(coming soon)





## Troubleshooting

```
## general ##
# EVENTS: kubectl get events -n <NAMESPACE>

## git-sync ##
# LOGS: kubectl logs -f <POD NAME> -c git-sync
# EXEC: kubectl exec -it <POD NAME> -c git-sync /bin/sh
# DOCKER: docker run --rm -it \
    -v /tmp/git-data:/tmp/git --entrypoint /bin/sh \
    k8s.gcr.io/git-sync:v3.1.5

## hugo ##
# LOGS: kubectl logs -f <POD NAME> -c hugo
# EXEC: kubectl exec -it <POD NAME> -c hugo /bin/sh
# DOCKER: docker run --rm -it \
    -v /tmp/git-data:/tmp/git --entrypoint /bin/sh \
    klakegg/hugo:0.65.2-ext-alpine
    
## nginx ##
# LOGS: kubectl logs -f <POD NAME> -c nginx
# EXEC: kubectl exec -it <POD NAME> -c nginx /bin/sh
# CONFIG: kubectl edit cm nginx-conf -o yaml
```

## Change version of hugo

Just change the hugo image tag in `examples/base/gitdocs-deployment.yaml`. For example: klakegg/hugo:**0.65.2**-ext-alpine. More info: https://github.com/klakegg/docker-hugo

## Thanks

A lot of this is built on top of the example present in the git-sync repo: https://github.com/kubernetes/git-sync/tree/master/demo
