# note to self 

1. Make sure the VM's are defined with a processor type X86-64-V2 or higher
2. Create 4 worker nodes at the minimum
   - Each with enough HD space to host the s3 buckets you need
   - Beware that you will need twice the required space as we will define 2 storage blocks for Minio
3. When the worker nodes are booted and ou have ssh access:
   - log in on each worker node and create two directories:
```
sudo mkdir -p /export0
sudo mkdir -p /export1
sudo chmod 744 /export0
sudo chmod 744 /export1
```
_Although the name can be choosen as you wish, make sure you remember them. Make sure they are not inside a user folder, like /home/bastiaan/_

4. Deploy the cluster with k03s or any other deployement tool
```
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-metastore
  user: admin
spec:
  hosts:
  - ssh:
      address: 192.168.180.155
      user: *** insert username you want to use ***
      port: 22
    role: controller
  - ssh:
      address: 192.168.180.156
      user: *** insert username you want to use ***
      port: 22
    role: worker
  - ssh:
      address: 192.168.180.157
      user: *** insert username you want to use ***
      port: 22
    role: worker
  - ssh:
      address: 192.168.180.158
      user: *** insert username you want to use ***
      port: 22
    role: worker
  - ssh:
      address: 192.168.180.159
      user: *** insert username you want to use ***
      port: 22
    role: worker    
  k0s:
    config:
      apiVersion: k0s.k0sproject.io/v1beta1
      kind: Cluster
      metadata:
        name: k0s
      spec:
        api:
          k0sApiPort: 9443
          port: 6443
        installConfig:
          users:
            etcdUser: etcd
            kineUser: kube-apiserver
            konnectivityUser: konnectivity-server
            kubeAPIserverUser: kube-apiserver
            kubeSchedulerUser: kube-scheduler
        konnectivity:
          adminPort: 8133
          agentPort: 8132
        network:
          kubeProxy:
            disabled: false
            mode: iptables
          kuberouter:
            autoMTU: true
            mtu: 0
            peerRouterASNs: ""
            peerRouterIPs: ""
          podCIDR: 10.244.10.0/16
          provider: kuberouter
          serviceCIDR: 10.96.10.0/12
        podSecurityPolicy:
          defaultPolicy: 00-k0s-privileged
        storage:
          type: etcd
        telemetry:
          enabled: true
```
5. When deployed login and create the default local file storage class  (it misses in k03s)
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner # indicates that this StorageClass does not support automatic provisioning
volumeBindingMode: WaitForFirstConsumer
```
https://kubernetes.io/docs/concepts/storage/storage-classes/#local

6. Now we create 4 Persistent Volumes, 2 on each worker node, bound to the directories we created above
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wn1-02  # the name of the Persitent Volume, I use a reference to the worker node nr.
spec:
  capacity:
    storage: 8Gi # make sure it is big enough for you bucket. 8 Gigabyte is small.. 
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage  # here we refere back to the storage class as created above 
  local:
    path: /export1  # this is the reference to the directory we created in step 3
  nodeAffinity:  # this Volume will be directly bind to a worker node
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - k3s-wna1  # the host name of your worker node
```
