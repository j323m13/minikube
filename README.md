# Pihole on minikube
Installation of pihole on minikube (demo)

Recently I was asked to prepare a short K8s demo. 
In this short demo, we will 
- configure 3 virtual machines (nodes) on minikube
- create our yaml configuration files to prepare our storage space, pods and the service that will map our pods to the main server (node). 
- launch an instance of pihole (ad blocker and dns server) (to see if everything works)
- scaling up and down our pihole instance (pods)
- Lesson learned

## Configuration: set up of three nodes in minikube
We assume that minikube, docker and kubenetes are installed on the machine. 

Otherwise (for MacOS): ``` brew install minikube ```. Follow the guide for installing Docker on MacOS: [link](https://docs.docker.com/desktop/mac/install/)

In the terminal, we create with minikube 3 servers (3 nodes) with the following options
- CPU: 2
- RAM: 2G
- Driver: virtualmachine
- p: server (server, server-m02, server-m03). m stands for multinode. 


 `minikube start --driver=virtualbox --cpus=2 --memory=2G --nodes=3 -p server`
 
 We have tested also different drivers options (hyperkit, docker) but the virtualbox driver yield the best result on our machine. 
 
 `minikube start --driver=hyperkit --cpus=2 --memory=2G --nodes=3 -p server`
 
  `minikube start --driver=docker --cpus=2 --memory=2G --nodes=3 -p server`
  
  
  we can test if our nodes have been created:
   `kubectl get nodes -o wide`
   
  For our exemple, we need the ip of our minikube server profile:
   `minikube ip -p server`
   We keep this ip for further use (in config file).

## set up our kubernetes cluster
As we can see in the docker image of pihole, we need to map the `53` ports (`UDP` and `TCP`) as well as the `8000` port (access to admin pannel: web interface). We also need to create two folder on our machine (`etc/` and `/dnsmasq.d`) and to map those folder inside the pods (`/etc/pihole` and `/etc/dnsmasq.d`). Last but not least, we need to set a password to access the web admin pannel. There are advantages and disadvantages to doing this. 
- Advantages: you can change the password and redeploy the pods. Each pod has the same password. 
- Disadvantage: if someone has access to the file, then they know the password. 

```
docker run -d \
  --name pihole \
  -p 53:53/tcp -p 53:53/udp \
  -p 8000:80 \
  -e TZ="America/New_York" \
  -e WEBPASSWORD="secret" \
  -v /path/to/volume/etc/:/etc/pihole/ \
  -v /path/to/volume/dnsmasq.d/:/etc/dnsmasq.d/ \
  --dns=0.0.0.0 --dns=1.1.1.1 \
  pihole/pihole:latest \
  ```
  ### YAML is life
  We will create the following files in yaml.
  1. `pihole.yaml` (where we set up the StorageClass, the PersistentVolumClaim, the PersistenVolume)
  2. `pihole-deployment.yaml` (where we set up the deploymwent option of our pihole app)
  3. `pihole-svc.yaml` (where we set up the service for our cluster)

#### `pihole.yaml`
Remember the ip we searched in minikube. We will set it here in `ExternalIPs`-Option:


```apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pihole-local-etc-volume
  labels:
    directory: etc
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local
  local:
    path: /Users/jeremieequey/Documents/programmation/workspace/docker/pihole-final/etc
  nodeAffinity:
    required:
      nodeSelectorTerms: # on which node we want to mount our volume
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - server
          - server-m02
          - server-m03
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pihole-local-etc-claim
spec:
  storageClassName: local
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      directory: etc
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pihole-local-dnsmasq-volume
  labels:
    directory: dnsmasq.d
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local
  local:
    path: /Users/jeremieequey/Documents/programmation/workspace/docker/pihole-final/dnsmasq.d
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - server 
          - server-m02 
          - server-m03
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pihole-local-dnsmasq-claim
spec:
  storageClassName: local
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      directory: dnsmasq.d

 ```

#### `pihole-deployment.yaml`


```apiVersion: apps/v1
kind: Deployment
metadata:
  name: pihole
  labels:
    app: pihole
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 1
  selector:
    matchLabels:
      app: pihole
  template:
    metadata:
     labels:
       app: pihole
       name: pihole
    spec:
      affinity:
        #This ensures pods will land on separate hosts
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions: [{ key: app, operator: In, values: [pihole] }]
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: pihole
        image: pihole/pihole:latest
        imagePullPolicy: Always
        env:
        - name: TZ
          value: "Europe/Zurich"
        - name: WEBPASSWORD
          value: "password"
        volumeMounts:
        - name: pihole-local-etc-volume
          mountPath: "/etc/pihole"
        - name: pihole-local-dnsmasq-volume
          mountPath: "/etc/dnsmasq.d"
      volumes:
      - name: pihole-local-etc-volume
        persistentVolumeClaim:
          claimName: pihole-local-etc-claim
      - name: pihole-local-dnsmasq-volume
        persistentVolumeClaim:
          claimName: pihole-local-dnsmasq-claim
      terminationGracePeriodSeconds: 0 # kill pods when terminating
 ```

#### `pihole-svc.yaml`


```apiVersion: v1
kind: Service
metadata:
  name: pihole
spec:
  selector:
    app: pihole
  ports:
    - port: 8000
      targetPort: 80
      name: pihole-admin
    - port: 53
      targetPort: 53
      protocol: TCP
      name: dns-tcp
    - port: 53
      targetPort: 53
      protocol: UDP
      name: dns-udp
  externalIPs:
  - 192.168.99.153 #assign our pihole-svc to a static ip (minikube ip -p server)
 ```

## Let's create
Now that we have our configuration files, we have to create the storage (pvc, pv), the pods, the deployment and the service in kubernetes:

1. `kubectl create -f pihole.yaml`
We check if the storage claim (pvc) and storage have been properly set:
`kubectl get pvc` and
`kubectl get pv`

2. `kubectl create -f pihole-deployment.yaml`
We check if the pods have been created:
`kubectl get pods -o wide`


3. `kubectl create -f pihole-svc.yaml`
We check if our service pihole is running:
`kubectl get service`

We check the logs (recursively -f) in a second terminal windows/tab:
`kubectl logs -f -l name=pihole`


## Test the configuration
We test if our dns server is reachable
`dig @IP-CLUSTER google.com`

We test if we can reach the admin web interface:
`IP-CLUSTER:8000/admin`

On our machine, we set up pihole as first dns server and test if web queries are succesful or not
`dig @IP-CLUSTER google.com` or directly with the web interface.

## let's scale it
We have 2 intances of pihole running on 2 separated pods (and maybe separated servers). We want now to scale it up to 5 pods:
`kubectl scale --current-replicas=2 --replicas=5 -p server`.

Keep in mind that this command does not survive a reboot. 

We check if we have 5 pods running:
`kubectl get pods -o wide`

Now we want only 1 pods on server-m02 and 1 pods (as a backup) running only on server-m03:
`kubectl delete pod NAME_OF_THE_PODS`
if the pods are not dying:
`kubectl delete pod --grace-period=0 --force PODNAME`

we create a yaml-file with our backup pod configuration which stipulates that it will only run on server-m03:

`pihole-backup-node`

```apiVersion: v1
kind: Pod
metadata:
  name: pihole-backup
  labels:
    app: pihole
spec:
  nodeName: server-m03
  containers:
    - name: pihole
      image: 'pihole/pihole:latest'
      imagePullPolicy: Always
      env:
        - name: TZ
          value: Europe/Zurich
        - name: WEBPASSWORD
          value: password
      volumeMounts:
        - name: pihole-local-etc-volume
          mountPath: /etc/pihole
        - name: pihole-local-dnsmasq-volume
          mountPath: /etc/dnsmasq.d
  volumes:
    - name: pihole-local-etc-volume
      persistentVolumeClaim:
        claimName: pihole-local-etc-claim
    - name: pihole-local-dnsmasq-volume
      persistentVolumeClaim:
        claimName: pihole-local-dnsmasq-claim
  terminationGracePeriodSeconds: 0
  ```

`kubectl create -f pihole-backup-node.yaml`

We check if the creation of this pod has been succesful:
`kubectl get pods -o wide`

Alrighat, now we have one pod running on server-m02 and another one running always on server-m03. Our installation should be safe for a while. 

## Lesson learned

+ wonderful learning effect
+ kubernetes is everywhere
+ well documented services
+ going further with try and error
- down into the rabbit hole
- yaml syntax will makes you cry
- going further with try and error


## how to improve our setup.
- the folderes `/etc` and `/dnsmasq.d` should be hosted on a separeted server, not on our machine.
- the web interface should be only accessible with ssl
- our setup should be accessible for all machines (or machines on a dedicated ip ranges)
- two instances of pihole that constantly synchronise (master-slave)


That's it for now. Take care. 








  
  
