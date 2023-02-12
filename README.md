# DEPLOYING MULTIPLE ARGOCD CLUSTERS WITH KIND https://github.com/tiacloudconsult/Tiacloud/tree/main/Documentations/ArgoCD #

# The purpose of this build is deploy a centralised but multiple argocd clusters. Hence rather than deploying argocd into its own aks cluster, we are endevouring to save costs. So we are creating  different argocd clusters using namespaces. Each ArgoCD cluster will have its own dedicated sub.domain.argocd.com. Moreover, we are deploying a management ArgoCD cluster which will self manage the ArgoCD repository.

# Deploy Kind multi-node cluster
--> cd kindcluster/kindcluster.yaml
--> kind create cluster --config kindcluster.yaml
--> kind export kubeconfig --name argocdclutser
--> kind export kubeconfig --name argocdcluster >> ~/.kube/config
--> kubectl config view

# Deploy Nginx controller
--> kubectl apply -f inginxingress.yaml

# Deploy MetalLB

--> cd ../metallb
--> kubectl apply -f metallb-ns.yml
--> kubectl apply -f metallb-manifests.yml

# Setup address pool used by loadbalancers
# To complete layer2 configuration, we need to provide metallb a range of IP addresses it controls. We want this range to be on the docker kind network.
--> docker network inspect -f '{{.IPAM.Config}}' kind
# The output will contain a cidr such as 172.19.0.0/16. We want our loadbalancer IP range to come from this subclass. We can configure metallb, for instance, to use 172.19.255.200 to 172.19.255.250 by creating the configmap.

--> vi lb-address-pool.yml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.18.0.30-172.18.0.60
      
--> kubectl apply -f lb-address-pool.yml

#ArgoCD Installation
# Download argocd cli -- For local machine (Mac, Windows, Linux)
--> https://argo-cd.readthedocs.io/en/stable/cli_installation/

# Create argocd namespace. In our case we are doing multiple ArgoCD clusters so create multiple namespaces
--> kubectl create namespace argocdeu #europe
--> kubectl create namespace argocdasia #asia
--> kubectl create namespace argocdna #northamerica
--> kubectl create namespace argocdmanager #the manager will be used to self manage argocd. Hence, Argocd
                                           #will be syncing its own repository 

# Deploy ArgoCD in each Namespace
--> cd ../
--> kubectl apply -f argocddeploy.yaml -n argocdmanager
--> kubectl apply -f argocddeploy.yaml -n argocdeu
--> kubectl apply -f argocddeploy.yaml -n argocdasia
--> kubectl apply -f argocddeploy.yaml -n argocdna

# Patch ArgoCD service to assign loadbalancer
--> kubectl patch svc argocd-server -n argocdeu -p '{"spec": {"type": "LoadBalancer"}}'
--> kubectl patch svc argocd-server -n argocdasia -p '{"spec": {"type": "LoadBalancer"}}'
--> kubectl patch svc argocd-server -n argocdna -p '{"spec": {"type": "LoadBalancer"}}'
--> kubectl patch svc argocd-server -n argocdmanager -p '{"spec": {"type": "LoadBalancer"}}'

# Deploy Ingress for ArgoCD

# To create your own certs, follow this documentation https://www.ibm.com/docs/en/api-connect/10.0.1.x?topic=overview-generating-self-signed-certificate-using-openssl

# Create secrets for Ingress TLS - secrets must be created for each ingress, you can use the same cert files
--> cd secret

kubectl create secret tls argocd-ingress-tls \
    --namespace argocdmanager \
    --key localhost.key \
    --cert localhost.crt 
    

kubectl create secret tls argocd-ingress-tls \
    --namespace argocdeu \
    --key localhost.key\
    --cert localhost.crt 

kubectl create secret tls argocd-ingress-tls \
    --namespace argocdna \
    --key localhost.key\
    --cert localhost.crt 

kubectl create secret tls argocd-ingress-tls \
    --namespace argocdasia \
    --key localhost.key\
    --cert localhost.crt 

kubectl create secret tls argocd-ingress-tls \   #This secret is for nginx-controller.
    --namespace default \                      #Secret is configure in line 424 of nginxingresscontroller.yaml
    --key localhost.key \
    --cert localhost.crt 

# Deploy Ingress
--> cd ../ingress
--> k delete ValidatingWebhookConfiguration ingress-nginx-admission  #bug, will not allow you to apply ingress.yaml files unless removed.
--> k apply -f argocdingress.yaml

# Edit /etc/hosts file and configure localhost (127.0.0.1 to point to newly created ingresses
--> vi /etc/hosts

127.0.0.1       manager.aifi.argocd.com 
127.0.0.1       asia.aifi.argocd.com
127.0.0.1       na.aifi.argocd.com
127.0.0.1       eu.aifi.argocd.com

# Access ArgoCD Dashboard
--> argoPassna=$(kubectl -n argocdna get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
--> argoPassasia=$(kubectl -n argocdasia get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
--> argoPasseu=$(kubectl -n argocdeu get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
--> argoPassmanager=$(kubectl -n argocdmanager get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# echo $argoPass<ns> to view the passwd to log into dashbaord. user: admin

# Add a Cluster to ArgoCD server

# Open a new Terminal create a new kind cluster ( as single-node cluster )
kind create cluster --name cluster01

# export and merge the kind cluster's kubeconfig to your local kubeconfig
kind export kubeconfig --name cluster01 >> ~/.kube/config

# Get the endpoint of the cluster you want to add to ArgoCD server
kubectl get endpoints
# ouput
NAME         ENDPOINTS           AGE
kubernetes   192.168.64.2:6443   37h

# Find the entry belonging to the cluster in your .kube/config, and change the server entry:
- cluster:
    certificate-authority-data: ...
    server: https://192.168.64.2:6443 
  name: <thecontext>

# Use the argocd cluser context and perform the following
kubectl config use-context kind-argocd 

# Add the cluster IP to the ArgoCD server
--> argocd cluster add kind-cluster01  

# Create ArgoCD Project
# create a project named cluster01-project. This can also be done in the ArgoCD UI
--> argocd proj create cluster01-project --description "this project is to deploy only cluster01 applications"

# Add Repository to ArgoCD. This can be also done in the ArgoCD UI
# NOTE: ONLY USE THIS IF YOUR APPLICATIONS ARE IN A PRIVATE GIT REPO
# Use your git repo with correct ssh path configured for the git account
--> argocd repo add git@github.com:<your-repo>.git --ssh-private-key-path ~/.ssh/id_rsa
# Create an ArgoCD Application
```yaml
# After deploying the app1, you need to register it as an applicaiton in ArgoCD server to monitor it

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app                                          # change to preferred name
  namespace: argocd
spec:
  destination:
    namespace: my-app                                   # change to preferred namespace
    server: https://192.168.64.2:6443                   # This server IP is for cluster01 
  project: default 
  source: 
    path: apps/bgd/overlays/bgd                         # change to path of deployment manifests in github repo
    repoURL: https://github.com/<your-github-repo>/     # Change to correct repo
    targetRevision: HEAD
  syncPolicy: 
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true


Configure Cluster
___Run the below commands on target cluster
kubectl apply -f configure-sa.yml
export serverurl=$(kubectl config view --minify -o jsonpath={.clusters[0].cluster.server})
export secrettoken=$(kubectl get serviceAccounts argocd-manager -n kube-system -o=jsonpath={.secrets[*].name})
export token=$(kubectl get -n kube-system secret/$secrettoken -o jsonpath='{.data.token}' | base64 --decode)
export ca=$(kubectl get -n kube-system secret/$secrettoken -o jsonpath='{.data.ca\.crt}')
___update configure-cluster.yml file with relevant details
___Run below command on Argocd Cluster 
kubectl apply -f configure-cluster.yml

argocdmanager_passwd=N5Y6MlG6ieIOBu1U