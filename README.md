# homelab-gitops-disaster-recovery
A means of bootstrapping a homelab in case of complete cluster failure, hardware migration, or even a fresh install. This is highly tailored to my own application, but should be easily extensible. 

Restoring a GitOps Kubernetes environment has a chicken-and-egg problem - how can applications be redeployed without the CD infrastructure used to deploy them? The approach taken here is to restore the bare minimum required to deploy applications - ArgoCD, Gitea, and Longhorn - and use that infrastructure to restore the remainder of the cluster from backups. 

## Disaster Recovery 

Argo in installed first, and used it to install applications defined here. Next, external longhorn volume backups are restored. Finally, Gitea is restored and all application resources can be deployed.

Using Argo instead of Helm has the benefit of correct labels on install, which prevents applications from being stuck out of sync due to Helm labels. 

### Prerequisites 

In order to recover from cluster failure or migrate to a new cluster, the following is required:

* A fresh kubernetes cluster
* Control of domain name. DDNS, routing, and DNS set up to route traffic to the cluster
* The Kubernetes admin token

### Playbook

1. Install k3s on the cluster using [k3s-ansbile](https://github.com/k3s-io/k3s-ansible), then run the `site` playbook:
    ```sh
    ansible-vault create vault-globals.yml
    ansible-vault edit vault-globals.yml 
    # Add `k3s_token: theTokenHere` to the vault
    ansible-playbook -i inventory.yml playbooks/site.yml --ask-vault-pass -e @vault-globals.yml
    ```
1. Install ArgoCD on the cluster with Helm
    ```sh
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update
    helm upgrade --install argo argo/argo-cd --version 8.1.3 -n argo --create-namespace
    ```
1. Clone this repo and `cd` into it. Alternatively, copy `application.yaml` from this repo's root
1. Apply the disaster recovery app-of-apps resource manifest `application.yaml`
    ```sh
    kubectl apply -f application.yaml
    ```
1. Connect to the ArgoCD GUI and sync the app-of-apps
    ```sh
    kubectl port-forward service/argo-cd-server -n argo-cd 8080:443
    # open the browser on http://localhost:8080 and accept the certificate
    ```
1. Sync the longhorn and longhorn-sc apps. Connect to the Longhorn GUI and restore all volumes
    ```sh
    kubectl port-forward service/longhorn-frontend -n longhorn-system 8081:80
    # open the browser at http://localhost:8081. Retrieve admin password with:
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```

