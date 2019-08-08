# Step-by-step guide: (Semi-)Production ready Kubernetes cluster with HA Storage using StorageOS.

It can be hard to create a small and simple High Available Kubernetes cluster which also has high available storage.
In this (hopefully) short guide, I'll try to explain my take on the way to do it.

## Requirements:
- At least, 3 Working machines that are going to act as both Kubernetes Masters and Workers.
Ofcourse you can vary this, but the guide will be build around this assumption. If you don't want to install storage nodes on your masters, you can just add workers, and skip the taint removal further on.

- A Loadbalancer for port 6443 (Kubernetes API)
- Some (free, duh) storage available on the machine.


## Step 0: Install Prerequisites


On *all* machines install:

- Docker
- Kubeadm, Kubectl, Kubelet
I use Ansible with the geerlingguy.docker and geerlingguy.kubernetes roles, they work fine.

- CNI Binaries
`curl -fsSL 'https://github.com/containernetworking/plugins/releases/download/v0.8.1/cni-plugins-linux-amd64-v0.8.1.tgz' | tar xvz -C /opt/cni/bin`
These are required for Weave networking.

## Step 1: Create Loadbalancer

You just need a simple loadbalancer for a single port (6443) to have a HA endpoint which connects to all running Kubernetes API servers. The loadbalancing makes sure your client (and external clients) can *always* talk to the API,regardless which node(s) are online.
If you skip this step, you will have 1 node which acts as the main API endpoint for your client.
If that node goes down, you have to manually point your client to another host, which can be hard and sometimes result in some certificate errors. So in this case i'm using the loadbalancer with an attached (internal) DNS entry, which means I can just use that url to connect to my Kubernetes cluster.

For my cluster I'm using pfSense. This is a FreeBSD based router distribution which is perfectly capable of the task!
It has builtin support for `haproxy` which is a really easy to configure loadbalancer package.

To install `haproxy` go to: `System` -> `Package Manager` -> `Available Packages`, Select `haproxy` and click `Install`.

Once installed go to: `Firewall` -> `Virtual IP's` and click `Add`. Select the proper `Interface` and set the IP Address you want to use for your loadbalancer.

If you want DNS support (recommended), go to `Services` -> `DNS Resolver` and click `Add` at the bottom of the page at the `Host Overrides` Section. In the 'Host' field choose something like `k8s`, and in the domain field, enter your local DNS domain (can be anything you want), in the example I'm using `prv.cloud`.
Set the `IP Address` to the ipaddress you chose in the `Virtual IP's` section.

Apply the changes and verify that you get the proper DNS result:

```bash
$ nslookup k8s.prv.cloud
Server:         192.168.1.1
Address:        192.168.1.1#53

Name:   k8s.prv.cloud
Address: 192.168.1.50
```

You can test it using ping, pfSense will respond by default:

```bash
$ ping -c 4 k8s.prv.cloud
PING k8s.prv.cloud (192.168.1.50) 56(84) bytes of data.
64 bytes from k8s.prv.cloud (192.168.1.50): icmp_seq=1 ttl=64 time=0.267 ms
64 bytes from k8s.prv.cloud (192.168.1.50): icmp_seq=2 ttl=64 time=0.253 ms
64 bytes from k8s.prv.cloud (192.168.1.50): icmp_seq=3 ttl=64 time=0.367 ms
64 bytes from k8s.prv.cloud (192.168.1.50): icmp_seq=4 ttl=64 time=0.352 ms

--- k8s.prv.cloud ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 74ms
rtt min/avg/max/mdev = 0.253/0.309/0.367/0.054 ms
```

## Step 1.1 - Setup HAProxy (still optional)

Now we need to configure HAProxy to loadbalance port 6443 between the kubernetes cluster nodes.

First we need a *Backend* as a set of target servers to point to.
Goto `Services` -> `HAProxy` -> `Backend`. Click `Add`.
Enter a `Name`, and in the `Server List` box click the tiny green arrow to add servers to the pool.
Each server needs a `Name`, `Address` and `Port`. In my case I have 3 entries, one for each server, with their respective IP Addresses and port 6443.

Click the `+` sign at the `Loadbalancer  options` section. Select `Round Robin`.
At the `Health checking` section, set the `Health check method` to `Basic`. This will allow the loadbalancer to forward traffic as soon as the TCP port is opened, it's fine for a basic setup, theoretically you can set an advanced healthcheck setup, but it's usually not required.

Click `Save` at the bottom to save the `Backend`.

Now we need to configure a *Frontend* to listen to port 6443 on the Virtual IP.
Click `Add` to create a Frontend. This is opens a port on your Virtual IP to start loadbalancing.

In the `Name` field, enter something meaningful. In the `External address`, under the `Listen address` dropdown, select the Virtual IP you created earlier. In the `Port` field, enter `6443`.
Because we need the Kubernetes API to handle the HTTPS part, set the `Type` to `TCP`. You could let pfSense handle the HTTPS traffic, but that's a bit harder to setup.

At the `Default backend` section set the `Default backend` to the backend you just created.
Click `Save` at the bottom to save the frontend.

Protip:
If you want to configure another port to use for an Ingress Controller repeat all the steps to create another Frontend and Backend, but at the Ports, enter `80` for HTTP or `443` for HTTPS.

## Step 1.2 - Test HAProxy (still optional)

To test the loadbalancer we need to spin up a container which listens on port 6443 on one of the nodes.
You can do something like:
`docker run -p 6443:80 nginx` and then use `curl k8s.prv.cloud:6443` to test the loadbalancer.

This will result in a 404 from Nginx if all works correctly.

## Step 2: Spinup cluster

We are going to use Kubeadm to spinup a cluster using a kubeadm config file. My current cluster file:

```yml
---
apiVersion: kubeadm.k8s.io/v1beta2
apiServer:
  timeoutForControlPlane: 4m0s
kind: ClusterConfiguration

certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
dns:
  type: CoreDNS
imageRepository: k8s.gcr.io

kubernetesVersion: v1.15.1

controlPlaneEndpoint: "k8s.prv.cloud:6443"

networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12

etcd:
  local:
    dataDir: /var/lib/etcd
```

This will spinup a so called 'stacked' kubernetes cluster. This means every master is also an Etcd cluster member.

Copy the file to one of the nodes, and execute the command:
`kubeadm init --config=kubeadm-config.yaml --upload-certs --v=2`
The --config refers to our config, the --upload-certs uploads the locally generated certificates to the cluster, and let's other masters download them upon joining. They will automatically be deleted after 2 hours for security purposes, so join your other masters within 2 hours. The command will explain this, and you can join masters afterwards, but you have to re-upload the certificates with a specific command.
The --v=2 gives us more logging.


If all goes well, you will see instructions on how to copy the kubectl config file and two long kubeadm commands to join a "Control Plane" node, or a normal worker. Use the copy command: `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config` to setup your connection to the cluster with kubectl.

Test your connection to your cluster using: `kubectl get nodes`. You will see 1 node, and it's status will be NotReady. That's normal, we'll fix that later.

Afterwards, copy the long command for the control plane, and execute this on all your other masters.
When all goes well you will see something like:

```bash
# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
nuc1   NotReady master   75m   v1.15.1
nuc2   NotReady master   73m   v1.15.1
nuc3   NotReady master   73m   v1.15.1
```

If you want to setup kubectl on all your nodes, execute the same cp command on all the other masters.

Go back to your first master, and execute the following command to install Kubernetes Networking:
`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`
This will install Weave Networking. Weave is a networking layer for Kubernetes which uses the CNI standard to provide inter-host communication for containers. It's required for the basic operation of Kubernetes.

You can monitor the installation using:
`kubectl get pod -n kube-system | grep weave`

When all 3 are running, you can verify weave is working correctly by looking at the CoreDNS containers using:
`kubectl get pod -n kube-system | grep dns`. If these are being started, or are already running, Kubernetes has detected weave as a CNI provider, and starts the CoreDNS containers using Weave's networking.

Wait until all pods in your cluster have reach the Running status.

## Step 3: Install StorageOS

We are going to use StorageOS as our Storage Provider. To read more about the cool features StorageOS has, go to their website: (https://www.storageos.com).

StorageOS runs a container per node, which acts as a layer between containers and the storage on the node.
These nodes form a cluster which makes sure the storage is kept in sync across multiple nodes.

## Step 3.1: Install Etcd

These nodes require a shared etcd cluster to keep their configuration in sync. So first we are going to install this Etcd cluster. The guys over at https://www.coreos.com have written a cool Kubernetes operator, which makes running an Etcd cluster on top of kubernetes super easy.

To install it, download their git repo:
`git clone https://github.com/coreos/etcd-operator.git`
Go into the directory: `example/rbac`, and execute `create_rbac.sh`, this will create a default RBAC setup for an Etcd cluster in the default namespace. If you want to change the namespaces, you can pass extra arguments to the command. (Docs: https://github.com/coreos/etcd-operator/blob/master/doc/user/rbac.md).

To deploy the etcd-operator, create a file called `storageos-operator.yml` with the following contents:

```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: etcd-operator
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: etcd-operator
    spec:
      containers:
      - name: etcd-operator
        image: quay.io/coreos/etcd-operator:v0.9.4
        command:
        - etcd-operator
        env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

Use `kubectl apply -f storageos-operator.yml` to deploy it.

Now we can create our Etcd cluster for StorageOS.

Create a file called `storageos-etcd-cluster.yml` with the following contents:

```yml
---
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "storageos-etcd"
spec:
  size: 3
  version: "3.3.13"
  pod:
    etcdEnv:
    - name: ETCD_QUOTA_BACKEND_BYTES
      value: "8589934592"  # 8 GB
    - name: ETCD_AUTO_COMPACTION_MODE
      value: "revision"
    - name: ETCD_AUTO_COMPACTION_RETENTION
      value: "100"
```

This will create a custom resource that the etcd-operator will pickup (if all goes well).
You can watch the cluster creation process using:
`kubectl get pod | grep etcd`.

If all goes well, you should see something like:

```bash
$ kubectl get pod | grep etcd
etcd-operator-866875d5dc-pchmv   1/1     Running   0          96m
storageos-etcd-7m6726t62f        1/1     Running   0          93m
storageos-etcd-jmgbpj2gzp        1/1     Running   0          94m
storageos-etcd-krr4ppvgg6        1/1     Running   0          94m
```

## Step 3.2 Install StorageOS Operator

Afterwards, we can deploy the StorageOS Cluster Operator. This operator will launch the StorageOS container on all (matching) nodes automatically, and makes sure, that when containers fail, appropriate actions are taken.

The operator will require some authentication, we can set this up using a secret.
Create a file called `storageos-api-secret.yml` with the following contents:
```yml
apiVersion: v1
kind: Secret
metadata:
  name: "storageos-api"
  namespace: "default"
  labels:
    app: "storageos"
type: "kubernetes.io/storageos"
data:
  # echo -n '<secret>' | base64
  apiUsername: c3RvcmFnZW9z
  apiPassword: c3RvcmFnZW9z
```
The values you see represent `storageos` in base64 encoded form. If you want to change these, run `echo -n 'yournewpassword' | base64` and replace the values in the file accordingly.

Apply the file with `kubectl apply -f storageos-api-secret.yml`.


Now, to deploy the StorageOS Operator, execute:
`kubectl create -f https://github.com/storageos/cluster-operator/releases/download/1.3.0/storageos-operator.yaml`

You can watch the process with `kubectl get pod -n storageos-operator`.

This will launch the StorageOS Operator on the cluster, which will look for the StorageOS Cluster resource. So to make the Operator actually work, we need to create a cluster.

Create a file called `storageos-cluster.yml` with the following contents:

```yml
apiVersion: "storageos.com/v1"
kind: StorageOSCluster
metadata:
  name: "storageos"
spec:
  secretRefName: "storageos-api" # Reference from the Secret created in the previous step
  secretRefNamespace: "storageos-operator"  # Namespace of the Secret
  images:
    nodeContainer: "storageos/node:1.3.0"
  kvBackend:
    address: 'storageos-etcd.default:2379' # Example address, change for your etcd endpoint
    backend: 'etcd'
  csi:
    enable: true
    deploymentStrategy: "deployment"
  resources:
    requests:
      memory: "512Mi"
```

This will create a basic cluster which uses our `storageos-api` secret, uses our `storageos-etcd` cluster, uses the basic CSI plugins, and reserves 512MB per node for the StorageOS Node Container.

You can monitor the startup of the cluster using: `kubectl get all -n storageos`.
When all pods (watch the 3/3) enter the "Running" state, the cluster is online.

# Step 3.3 Open the GUI
The StorageOS Operator comes with a WebGUI which listens on port 5705. So you have to first find out which node the operator was started on, and then you can open the webinterface.
On one of the masters, execute: `kubectl get pod -n storageos-operator -o wide`, this will show something like:

```bash
NAME                                          READY   STATUS    RESTARTS   AGE    IP          NODE   NOMINATED NODE   READINESS GATES
storageos-cluster-operator-68b7bddcbb-mxr5d   1/1     Running   0          106m   10.46.0.3   nuc1   <none>           <none>
```
Find the node where the operator is running. In my case, this tells me, that my pod is running on nuc1.

On your local workstation go to https://<ip/hostname-of-the-node>:5705.
If all went well, you will be greeted with an HTTPS warning, and after accepting this, a login page.
Use the username and password configured in the secret earlier. In my example it's `storageos` for both the username and the password.

To verify the pool is healthy, click `Pools` on the left. In this overview you can see how much storage is available to your cluster. Because we didn't setup any extra drives or mountpoints, StorageOS is just using the root filesystem for it's storage pool.

# Step 3.4 Provision a volume

To start using our persistent storage, we need to create a PersistentVolumeClaim (which is a Kubernetes default resource).

Create a file called `my-vol.yml` with the following contents:

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-vol
  labels:
    storageos.com/replicas: "2"
  annotations:
    volume.beta.kubernetes.io/storage-class: fast
spec:
  accessModes:
  - ReadWriteOnce
  resources:
   requests:
     storage: 5Gi
```

This will create a volume called my-vol, it will have 2 replica's (so it's high available!), will be scheduled on 'fast' storage, and will have a size of 5 gigs.

As usual, execute: `kubectl apply -f my-vol.yml` to create the PVC.

Now we can spin up a container that uses the volume!

Create a file called `test-my-vol.yml` with the following contents:

```yml
apiVersion: v1
kind: Pod
metadata:
  name: test-my-vol
spec:
  containers:
    - name: debian
      image: debian:9-slim
      command: ["/bin/sleep"]
      args: [ "3600" ]
      volumeMounts:
        - mountPath: /mnt
          name: v1
  volumes:
    - name: v1
      persistentVolumeClaim:
        claimName: my-vol
```
Execute: `kubectl apply -f test-my-vol.yml` to create the container.

This will create a simple debian container which uses the PVC.

To watch the container spinup, use: `kubectl get pod | grep test`.
Once it's `Running` you can enter the container using: `kubectl exec -it <podname> /bin/bash`.

Inside the container you can go to /mnt, and there you will have your storage!
