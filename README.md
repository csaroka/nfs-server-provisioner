# nfs-server-provisioner

## Enter kube-system namespace
`$ kubectl config set-context $(kubectl config current-context) --namespace=kube-system`

## Create a Default Storage Class
    
`$ kubectl apply -f default-sc.yaml`

## Install Helm Client
https://docs.helm.sh/using_helm/#installing-helm 
or 

`$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh`

`$ chmod 700 get_helm.sh`

`$ ./get_helm.sh`

## Create the Tiller RBAC Policy

`$ kubectl apply -f tiller-rbac.yaml`

## Initialize Tiller

`$ helm init --service-account tiller`

## Edit the chart values.yaml (custom override)
Note: Review the size of the volume and edit

## Install the NFS Server Provisioner Chart

`$ helm install stable/nfs-server-provisioner --name k8s-nfs-server -f values.yaml`


## Create the Namespace

`$ kubectl create ns corp-lab`
`$ kubectl config set-context $(kubectl config current-context) --namespace=corp-lab`


## Create Persistent Volume Claim

`$ kubectl apply -f corpweb-nfs-pvc.yaml`

##. Apply Deployment, Service, and Ingress (Note: Will fail until PVC contains data)

`$ kubectl apply -f corpweb-app.yaml`

##. Copy Web Content to NFS volume
`$ kubectl get pods`
`$ kubectl cp index.html <podname>:/usr/share/nginx/html/`
`$ kubectl exec -it <podname> /bin/bash`
`<pod>$ ls /usr/share/nginx/html/`


## Scale out Deployment

`$ kubectl scale --replicas=3 deploy/corpweb-deploy`


## Get Ingress Path

`$kubectl get ingress`

## Test Browser Access
`Example: http://corpweb.k8s01-app.lab.local`

