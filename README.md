# Internal Developer Platform (IDP)

A production-grade Internal Developer Platform built on Kubernetes, demonstrating GitOps-first Platform Engineering using [Backstage](https://backstage.io), [ArgoCD](https://argo-cd.readthedocs.io), and [Kind](https://kind.sigs.k8s.io).

## Bootstrapping k8s cluster

- `kind create cluster --config ./kind-envs/kind_cluster-IDP.yaml`\
    This command will create kind cluster on local machine.\
    https://kind.sigs.k8s.io/docs/user/quick-start/

- `kind delete cluster --name grzegorz-k8s-cluster`\
    This command will delete kind cluster.

## Bootstrapping argocd on k8s cluster

- Take from `https://github.com/argoproj/argoproj-deployments/blob/master/argocd/kustomization.yaml`, the kustomization.yaml and resources/namespace.yaml
- `kustomize build . | kubectl apply -f -` run it in kustomization.yaml dir or `k apply -k .`
- `k get po -n argocd`
- To port-forward argocd to local browser, use this command `k port-forward svc/argocd-server -n argocd 8085:80`
- To decode admin password for argocd UI, use `k get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`
- Then please run `k apply -f argocd-app.yaml -n argocd`