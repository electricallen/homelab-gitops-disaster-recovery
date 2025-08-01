# homelab-gitops-disaster-recovery
A means of bootstrapping a homelab in case of complete cluster failure, hardware migration, or even a fresh install. This is highly tailored to my own application, but should be easily extensible. 

Restoring a GitOps Kubernetes environment has a chicken-and-egg problem - how can applications be redeployed without the CD infrastructure used to deploy them? The approach taken here is to restore the bare minimum required to deploy applications - ArgoCD, Gitea, and Longhorn - and use that infrastructure to restore the remainder of the cluster from backups. 

## Disaster Recovery 

ArgoCD is installed first, and used to install the application manifests defined here. Next, external longhorn volume backups are restored. Finally, Gitea is restored and all application resources can be deployed.

ArgoCD only uses `helm` to produce manifests with `helm template`. This has the benefit of avoiding Helm labels on resources and prevents applications from being stuck out of sync. 

### Prerequisites 

In order to recover from cluster failure or migrate to a new cluster, the following is required:

* Hardware capable of running k3s, ideally with 3 server nodes
* Control of domain name. DDNS, routing, and DNS set up to route traffic to the cluster
* The Kubernetes admin token
* Backup restore credentials and URL
* Gitea admin credentials

### Playbook

1. Install k3s on the cluster by cloning [k3s-ansbile](https://github.com/k3s-io/k3s-ansible), setting up the inventory, then running the `site` playbook:
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
    helm upgrade --install argocd argo/argo-cd --version 8.1.3 -n argocd --create-namespace
    ```
1. Clone this repo and `cd` into it
1. Copy the bootstrap values file. Edit it to with backblaze and Gitea credentials (see [Longhorn docs here](https://longhorn.io/docs/1.9.0/snapshots-and-backups/backup-and-restore/set-backup-target/#set-the-default-backup-target-using-a-manifest-yaml-file))
    ```sh
    cp bootstrap/example-values.yaml bootstrap/values.yaml
    # Edit values.yaml to add backblaze and gitea creds
    ```
1. Use `helm` to template and apply the bootstrap manifest (Note: installing with helm is not advised because of labelling conflicts with ArgoCD)
    ```sh
    helm template bootstrap | kubectl apply -f -
    ```
1. Connect to the ArgoCD GUI and sync the Longhorn application. You may have to sync a couple times to get a healthy result.
    ```sh
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    # Note the admin password
    kubectl port-forward service/argocd-server -n argocd 8080:443
    # open the browser on http://localhost:8080 and accept the certificate
    ```
1. Connect to the Longhorn GUI
    ```sh
    kubectl port-forward service/longhorn-frontend -n longhorn-system 8081:80
    ```
1. Restore all volumes from Backblaze using the Longhorn GUI, choosing `Use Previous Name`
1. Navigate to the volumes tab and recreate PVs/PVCs

    ![alt text](docs/image.png)
    
1. Sync the Gitea application from the ArgoCD GUI
1. Connect to Gitea and validate repo contents have been restored
    ```sh
    kubectl port-forward service/gitea-http -n gitea 8082:3000
    ```