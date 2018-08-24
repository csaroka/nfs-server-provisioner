# nfs-server-provisioner

## Enter kube-system namespace
`$ kubectl config set-context $(kubectl config current-context) --namespace=kube-system`

## Create a Default Storage Class
`$ vi default-sc.yaml`

`kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: thin
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/vsphere-volume
parameters:
    diskformat: thin`
    
`$ kubectl apply -f default-sc.yaml`

## Install Helm Client
https://docs.helm.sh/using_helm/#installing-helm 
or 

`$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh`

`$ chmod 700 get_helm.sh`

`$ ./get_helm.sh`

## Create the Tiller RBAC Policy
`$ vi tiller-rbac.yaml`

`apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller`
    

`$ kubectl apply -f tiller-rbac.yaml`

## Initialize Tiller

`$ helm init --service-account tiller`

## Create the chart values.yaml (custom override)
Note: Review the size of the volume and edit

`$ vi values.yaml`

`# Declare variables to be passed into your templates.

replicaCount: 1

# imagePullSecrets:

image:
  repository: quay.io/kubernetes_incubator/nfs-provisioner
  tag: v1.0.9
  pullPolicy: IfNotPresent

service:
  type: ClusterIP

  nfsPort: 2049
  mountdPort: 20048
  rpcbindPort: 51413
  # nfsNodePort:
  # mountdNodePort:
  # rpcbindNodePort:

  externalIPs: []

persistence:
  enabled: true

  ## Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  storageClass: "thin"

  accessMode: ReadWriteOnce
  size: 120Gi

## For creating the StorageClass automatically:
storageClass:
  create: true

  ## Set a provisioner name. If unset, a name will be generated.
  # provisionerName:

  ## Set StorageClass as the default StorageClass
  ## Ignored if storageClass.create is false
  defaultClass: false

  ## Set a StorageClass name
  ## Ignored if storageClass.create is false
  name: nfs

## For RBAC support:
rbac:
  create: true

  ## Ignored if rbac.create is true
  ##
  serviceAccountName: default

resources: {}
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
  #  memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}`


## Install the NFS Server Provisioner Chart

`$ helm install stable/nfs-server-provisioner --name k8s-nfs-server -f values.yaml`


## Create the Namespace

`$ kubectl create ns corp-lab`
`$ kubectl config set-context $(kubectl config current-context) --namespace=corp-lab`


## Create Persistent Volume Claim

`$ vi corpweb-nfs-pvc.yaml`


`kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: app-pvc-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: nfs
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi`


`$ kubectl apply -f corpweb-nfs-pvc.yaml`

##. Apply Deployment, Service, and Ingress (Note: Will fail until PVC contains data)

`$ vi corpweb-app.yaml`


`---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: corpweb-deploy
  labels: 
    app: corpweb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: corpweb
  template:
    metadata:
      labels:
        app: corpweb
    spec:
      containers:
      - name: websvr
        image: nginx
        ports:
          - name: http
            containerPort: 80
        volumeMounts:
            # name must match the volume name below
            - name: appvol
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: appvol
        persistentVolumeClaim:
          claimName: app-pvc-claim
---
#Service
apiVersion: v1
kind: Service
metadata:
  name: corpweb-svc
#  namespace: corp-lab
  labels:
    app: corpweb
spec:
  ports:
  - port: 80
    # targetPort: 8080
  selector:
    app: corpweb  
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: corpweb-ing
  #namespace: corp-lab
  labels:
    app: corpweb
spec:
  rules:
  - host: corpweb.k8s01-app.lab.local
    http:
      paths:
      - backend:
          serviceName: corpweb-svc
          servicePort: 80`

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

