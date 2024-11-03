# Volumes

- Storing Data inside EKS is not a good thing. We will keep data outside of cluster
- Here, in kubernetes we have two options. They are:
    - EBS (Elastic Block Storage) 
    - EFS (Elastic File Storage)
- We need to create volume outside of cluster and mount that volume to the cluster

## 1. EBS 
- EBS is like external Harddisk. It is offline which is near to computer and has more speed
- EBS is of two types. They are:
    - Static Provisioning
    - Dynamic Provisioning

![alt text](../images/k8-volumes.drawio.svg)

**Static Provisioning**
- Here, static means everything we need to do.
    i. First, we need to create volumes
    ii. We need to install drivers
    iii. EKS nodes should have permission to access EBS Volumes 

- Suppose node is in us-east-1b and can we create volume disk in us-east-1d?
    - No, wherever the server is located, disk should also be in the same availability zone

- Go to volumes in EC2 instance and create 20GB volume and copy the volume ID (vol-0903636d1319edc3f)
- Then we need to install ebs-csi drivers (Administration Task)
    - Go to documentation
    ```
    https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md
    ```
    - Run the following command to install ebs-csi drivers
    ```
    kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.36"
    ```
    - Run the command to see the namespaces
    ```
    kubectl get namespaces
    ```
    - Here, we can see `kube-system` where the administration activities are available
    - We can see the pods running in that namespace
    ```
    kubectl get pods -n kube-system
    ```
    - In kubernetes, everthing is pod. We have installed drivers which will also run as pods
    
    - For every EC2 instance, there will be IAM Role. Go to instance and click on IAM Role and click on `attach policies`
    - Search for EBS and select `AmazonEBSCSIDriverPolicy` and click on `Add permissions`

**Persistant Volumes and Persistant Volume Claim**
- Kubernetes created a wrapper objects to manage underlying volumes because k8 engineer will not have full idea on volumes
- PV represents the physical storage like EBS/EFS
- If we operate PV then PV will access volumes. So, if we run something on PV then PV will go and access volumes

- managing storage is distinct problem from managing compute instances 
- The PersistantVolume subSystem provides an API for users and administrators that abstracts details of how storage is provided from how it is consumed

- Now, we need to create PV
- We have multiple AccessModes
1. ReadWriteOnce -> the volume can be mounted as read-write by a single node. ReadWriteOnce access mode still can allow multiple pods to access the volume when the pods are running on the same node.
2. ReadOnlyMany  -> the volume can be mounted as read-only by many nodes.
3. ReadWriteMany -> the volume can be mounted as read-write by many nodes.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-static
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 20Gi
  csi:
    driver: ebs.csi.aws.com
    fsType: ext4
    volumeHandle: vol-0903636d1319edc3f
```
- Here, capacity should be equivalent to the capacity created for volume. 
- Volume ID should be specified which we have created earlier

```
kubectl apply -f 01-ebs-static.yml
```
```
kubectl get pv
```

**RECLAIM POLICY**
**1. Retain:** When node is deleted, and we may not loose data
    - When PersistantVolumeClaim is deleted, the PersistantVolume still exists and the volume is considered "released"
**2. Delete:** Whenever we delete the PersistantVolume, the disk gets deleted 
**3. Recycle:** Disk will not delete but the data cleans up

- Till now we have created actual storage and created equivalent object PV in cluster. Now, we have storage in our cluster and pod should ask for storage its nothing but claiming. Pod should request kubernetes cluster for storage. It is called PVC (Persistant Volume Claim)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-static
spec:
  storageClassName: "" # Empty string must be explicitly set otherwise default StorageClass will be set
  volumeName: ebs-static
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
- Here, storageClassName should be kept empty for static provisioning
- volumeName should be matched.
- Storage should be less or equivalent to the volume we have created

```
kubectl apply -f 01-ebs-static.yml
```
```
kubectl get pvc
```
- Here, we can see the status as `bound` which means PVC has attached to PV
```
kubectl describe pvc ebs-static
```
```
kubectl describe pv ebs-static
```
- Create pod and attach volume through PVC
```
apiVersion: v1
kind: Pod
metadata:
  name: ebs-static
  labels:
    purpose: ebs-static
spec:
  nodeSelector:
    topology.kubernetes.io/zone: us-east-1d
  containers:
  - name: nginx
    image: nginx
    volumeMounts: # docker run -v hostpath:contaierpath
    - name: ebs-static
      mountPath: /usr/share/nginx/html
  volumes:
  - name: ebs-static
    persistentVolumeClaim:
      claimName: ebs-static
```
```
kubectl apply -f 01-ebs-static.yml
```
```
kubectl get pods
```
```
kubectl get pods -o wide
```
- Here, we need to specify nodeselector. We should say pod to go to that node.
```
kubectl get nodes --show-labels
```
- Delete the pod and recreate it

```
kubectl describe pod ebs-static
```
- Here, we can see that `AttachVolume.Attach succeded for volume "ebs-static"`

- Now, attach service to the pod and make sure allow security groups of particular nodeport

```
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    purpose: ebs-static
  ports:
  - name: nginx-svc-port
    protocol: TCP
    port: 80 # service port
    targetPort: 80 # container port
    nodePort: 30007
```

```
kubectl get pods
```
```
kubectl exec -it ebs-static -- bash
```
```
cd /usr/share/nginx/html/
```
```
echo "<h1>This is EBS Storage</h1>" > index.html
```
- Now, if we copy the LB URL and browse it in internet, we will get this index.html page
- If we delete pod and recreate it, the data will be there.
- This is static provisioning

**Key Points**
- PV is the physical representation of the storage. It is the wrapper object which represents the external storage
- PVC is the claim requesting for storage. PVC contains PV information
- POD gets the storage through PVC


**Dynamic Provisioning**
- Here, dynamic means everything will be taken care by kubernetes
- Volumes will be created by kubernetes

To view all the resources in kubernetes cluster
```
kubectl api-resources
```
- When we create any resource, we have two types
    - It will be namespace level
    - It will be cluster level
- Here, PV is the cluster level and admin has to create it and PVC is namespace level

**expense project devops engineer got a requirement to have a volume**
1. You should send an email to storage team to create disk, then get the approval from manager and after getting approval they will create disk 
2. You send an emain to K8 admin to create PV and provide them disk details

Now it's your turn is to create pvc and claim in the pod 
-> we have to ask for administartor for any changes in the permissions

- Now, delete the ebs-static pod and also volume created in AWS

- In dynamic provisioning, we no need to create volume and PV. 
- Default storageClass reclaimpolicy is delete. We don't use defalut
```
kubectl get sc
```
- We will create our own storage class (This is admin activity)
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expense-ebs
reclaimPolicy: Retain
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer 
```
- Here, storage will be created when pod is getting created
- Now, we will create dynamically 

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-dynamic
spec:
  storageClassName: "expense-ebs"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-dynamic
  labels:
    purpose: ebs-dynamic
spec:
  nodeSelector:
    topology.kubernetes.io/zone: us-east-1d
  containers:
  - name: nginx
    image: nginx
    volumeMounts: # docker run -v hostpath:contaierpath
    - name: ebs-dynamic
      mountPath: /usr/share/nginx/html
  volumes:
  - name: ebs-dynamic
    persistentVolumeClaim:
      claimName: ebs-dynamic
---
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    purpose: ebs-dynamic
  ports:
  - name: nginx-svc-port
    protocol: TCP
    port: 80 # service port
    targetPort: 80 # container port
    nodePort: 30007
```
- Here, kubernetes will create a volume automatically
```
kubectl get pv
```


## 2. EFS 
- EBS is like google drive. It is online which is somewhere in network and has less speed

**EBS vs EFS**
1. EBS is a block storage, EFS is like NFS (Network File System)
2. EBS should be as near as possible.EFS can be anywhere in the network 
3. EBS is fast campared to EFS 
4. EBS we can store OS, databases. EFS is not good OS and databases
5. EFS can increase stoarge limit automatically based on requirement  
6. Generally files are stored in EFS (like customer related forms and signed documents)

- EBS is of two types. They are:
    - Static Provisioning
    - Dynamic Provisioning

**Static Provisioning**
- what are the things we have to provide in static provisioning ?
1. Create EFS volume 
2. Install drivers and allow 2049 traffic from EKS worker nodes 
3. Give permissions to EKS nodes 
4. Create PV
5. Create PVC 
6. Claim through pod using PVC 
7. Open node port in the EKS worker nodes 

- First we will create EFS volume. Go to EFS in AWS console and create volume in `eksctl-expense-1-cluster/vpc` location
- Then, install efs-csi drivers from the following location
```
https://github.com/kubernetes-sigs/aws-efs-csi-driver
```
- Use the following command to install drivers and we need to change the release version here. We will take public registry
```
kubectl kustomize \
    "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-2.X" > public-ecr-driver.yaml
```
- This command output will be saved in the file named `public-ecr-driver.yaml` and run the command on this file
```
kubectl apply -f public-ecr-driver.yaml
```

![alt text](../images/k8-volumes-EFS.drawio.svg)

- Ultimately requests comes from nodes to the volumes
- If the request is from pod also that comes out of the node to the NFS 
- For NFS, 2049 is the protocol 

- Now, EFS SecurityGroup should allow traffic on 2049 from the SecurityGroup attached to EKS worker nodes
    - Open EFS file system which was created earlier and open network and copy security group ID
    - Then open EFS securitygroup and click on `edit inbound rules` and add `NFS` and provide `EKS securitygroup ID` and click on `save rules` 

- Now, go to instance AMI roles and click on `attach policies` and search for EFS and select `AmazonEFSCSIDriverPolicy` and click on `Add permissions`

- Now, we need to create PV (Persistant Volume)

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: expense-efs
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-020cccf0fbad56433
```
- Here, we need to provide volumeID which we have created in EFS

