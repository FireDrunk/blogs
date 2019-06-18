# Rancher Kubernetes HA Cluster setup

A few weeks back we got an *awesome* demo from the Chris (CEO) from StorageOS (https://storageos.com).
To try the software, I wanted to build a full HA Kubernetes cluster to test out the software.
This is the story about my endouvers to get a fully HA Rancher Cluster up and running.

# Environment
My homelab consists of 3 "mini" PC's. Specs are not very important, but they are all using SSD's for the root filesystem, and have an extra SSD attached for storing data. They are connected using basic Gigabit networking.

My homelab also features a pfSense router, which I'll be using as the external Load Balancer as recommended by Kubernetes. This also happens to be my home router, so I can (and will) expose the cluster to the internet in the future.

## Requirements:
For this setup to work, one would need:
- 3 Hosts (on the same network).

### Optional
- An external Load Balancer pointing towards the nodes.
- Extra devices on the nodes for extra storage.

# Step 1 - Installation of the Hosts
As I'm using baremetal machines, there is no automatic installation of the base operating system. Of course you could do something with PXE boot, or something fancy with automated provisioning, but that's not the story here.

I'm going to use Ansible to manage these machines, so a bare install is the only requirement, the rest will be done with Ansible.

I've manually installed all three machines with a minimal Ubuntu netinstall and setup an SSH account. I've generated a keypair for Ansible to connect to the machines, created accounts on the machines, and setup key authentication for these accounts. If you don't know how to do this, there are *tons* of guides on the internet ;)

The basic (directory) setup:

```bash
root@nas:/home/thijs/nuc-cluster# ls
total 32K
drwxrwxr-x 6 thijs thijs 4.0K Jun 18 08:04 .
drwxr-xr-x 8 thijs thijs 4.0K Jun 18 07:50 ..
-rw-r--r-- 1 thijs thijs  270 Jun 18 07:49 ansible.cfg
drwxrwxr-x 5 thijs thijs 4.0K Jun 18 07:49 applications
drwxrwxr-x 2 thijs thijs 4.0K Jun 18 11:47 basic
-rw-r--r-- 1 thijs thijs   52 Jun 18 07:49 inventory
drwxrwxr-x 2 thijs thijs 4.0K Jun 18 07:49 ssh
drwxrwxr-x 5 thijs thijs 4.0K Jun 18 11:50 storageos
```

`ansible.cfg` contains settings used by Ansible to connect to the cluster, for instance, the correct username, the location of the key, and settings regarding privilege escalation (ubuntu comes without direct root access out-of-the-box).
The `applications` directory contains deployment files for things we want to deploy on the cluster.
The `basic` directory contains the playbooks needed to install `Docker` and some core dependencies.
The `inventory` file contains the hosts in my network:

```bash
root@nas:/home/thijs/nuc-cluster# cat inventory
[k8s]
node1.cravu.lan
nuc2.cravu.lan
j4105.cravu.lan
```

The `ssh` directory contains the private key required to connect to the nodes.

```bash
root@nas:/home/thijs/nuc-cluster# ls ssh/
total 16K
drwxrwxr-x 2 thijs thijs 4.0K Jun 18 07:49 .
drwxrwxr-x 6 thijs thijs 4.0K Jun 18 08:04 ..
-rw------- 1 thijs thijs 2.6K Jun 18 07:49 provision
-rw-r--r-- 1 thijs thijs  564 Jun 18 07:49 provision.pub
```

The `storageos` directory contains kubernetes deployment files we'll use in another blog.

# Step 2 - Verify connection to all hosts

To test the Ansible setup you can use the following command:
`ansible all -i inventory -m ping`
This should result in something like:

```bash
root@nas:/home/thijs/nuc-cluster# ansible all -i inventory -m ping
nuc2.cravu.lan | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
nuc1.cravu.lan | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
j4105.cravu.lan | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

# Step 3 - Install Docker

To install Docker just run: `ansible-playbook basic/docker.yml`, this will install and enable Docker on the machines.

# Step 4 - Setup Load Balancer IP (optional)

You just need a simple loadbalancer for a single port (443) to have a HA endpoint which connects to all running Kubernetes API servers. The loadbalancing makes sure your client (and external clients) can *always* talk to the API,regardless which node(s) are online.

If you skip this step, you will have 1 node which acts as the main DNS for your client. If that node goes down, you have to manually point your client to another host, which can be hard and sometimes result in some certificate errors. So in this case i'm using the loadbalancer with an attached (internal) DNS entry, which means I can just use that url to connect to my Kubernetes cluster.

For my cluster I'm using pfSense. This is a FreeBSD based router distribution which is perfectly capable of the task!
It has builtin support for `haproxy` which is a really easy to configure loadbalancer package.

To install `haproxy` go to: `System` -> `Package Manger` -> `Available Packages`, Select `haproxy` and click `Install`.

Once installed go to: `Firewall` -> `Virtual IP's` and click `Add`. Select the proper `Interface` and set the IP Address you want to use for your loadbalancer.

If you want DNS support (recommended), go to `Services` -> `DNS Resolver` and click `Add` at the bottom of the page at the `Host Overrides` Section. In the 'Host' field choose something like `k8s`, and in the domain field, enter your local DNS domain (can be anything you want), in the example I'm using `cravu.lan`.
Set the `IP Address` to the ipaddress you chose in the `Virtual IP's` section.

Apply the changes and verify that you get the proper DNS result:

```bash
root@nas:/home/thijs/nuc-cluster# nslookup k8s.cravu.lan
Server:         192.168.1.1
Address:        192.168.1.1#53

Name:   k8s.cravu.lan
Address: 192.168.1.50
```

You can test it by pinging it, pfSense will respond by default:

```bash
root@nas:/home/thijs/nuc-cluster# ping -c 4 k8s.cravu.lan
PING k8s.cravu.lan (192.168.1.50) 56(84) bytes of data.
64 bytes from k8s.cravu.lan (192.168.1.50): icmp_seq=1 ttl=64 time=0.207 ms
64 bytes from k8s.cravu.lan (192.168.1.50): icmp_seq=2 ttl=64 time=0.237 ms
64 bytes from k8s.cravu.lan (192.168.1.50): icmp_seq=3 ttl=64 time=0.234 ms
64 bytes from k8s.cravu.lan (192.168.1.50): icmp_seq=4 ttl=64 time=0.247 ms

--- k8s.cravu.lan ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3054ms
rtt min/avg/max/mdev = 0.207/0.231/0.247/0.018 ms
```

# Step 4.1 - Setup HAProxy (still optional)

Now we need to configure HAProxy to loadbalance port 443 between the kubernetes cluster nodes.

First we need a *Backend* as a set of target servers to point to.
Goto `Services` -> `HAProxy` -> `Backend`. Click `Add`.
Enter a `Name`, and in the `Server List` box click the tiny green arrow to add servers to the pool.
Each server needs a `Name`, `Address` and `Port`. In my case I have 3 entries, one for each server, with their respective IP Addresses and port 443.

Click the `+` sign at the `Loadbalancer  options` section. Select `Round Robin`.
At the `Health checking` section, set the `Health check method` to `Basic`. This will allow the loadbalancer to forward traffic as soon as the TCP port is opened, it's fine for a basic setup, theoretically you can set an advanced healthcheck setup, but it's usually not required.

Click `Save` at the bottom to save the `Backend`.

Now we need to configure a *Frontend* to listen to port 443 on the Virtual IP.
Click `Add` to create a Frontend. This is 'opens' a port on your Virtual IP to start loadbalancing.

In the `Name` field, enter something meaningful. In the `External address`, under the `Listen address` dropdown, select the Virtual IP you created earlier. In the `Port` field, enter `443`.
Because we need Rancher to handle the HTTPS part, set the `Type` to `TCP`. You could let pfSense handle the HTTPS traffic, but that's a bit harder to setup.

At the `Default backend` section set the `Default backend` to the backend you just created.
Click `Save` at the bottom to save the frontend.


# Step 4.2 - Test HAProxy (still optional)

To test the loadbalancer we need to spin up a container which listens on port 443 on one of the nodes.
You can do something like:
`docker run -p 443:80 nginx` and then use `curl k8s.cravu.lan:443` to test the loadbalancer.
Be aware, this is not using https, but rather http, it will switch to https when Rancher is the one behind the port rather than a simple nginx container ;)

In my case the curl command results in:
```bash
root@nas:/home/thijs/nuc-cluster# curl k8s.cravu.lan:443
<html>
<head><title>400 The plain HTTP request was sent to HTTPS port</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>The plain HTTP request was sent to HTTPS port</center>
<hr><center>nginx/1.15.6</center>
</body>
</html>
```
But to be fair, my cluster is already working, so the nginx container is already from Rancher ;)

# Step 5 - Install Rancher (finally)

To install Rancher we could use RKE, but RKE lacks support for passwordless sudo (which Ansible supports flawlessly...). So instead, I'll be using the graphical installer.

To start the Rancher installer, login to one of your nodes, and run `docker run -d --restart=unless-stopped -p 443:443 rancher/rancher`. This will launch the Rancher installer on your first node.

It takes some time for the Rancher server to start, and for the loadbalancer (if you have one) to pick it up. Expect something like 10-15 seconds.

Now you can point your browser to your configured DNS name, in my case: (https://k8s.cravu.lan). If you've skipped the loadbalancer part, you can either use the DNS address or the IP Address of the node you started the Rancher installer on. Here you will be presented with the Rancher login screen. Choose a password, and click `Continue`.

You will now be prompted for your Cluster URL. If you've setup a Load Balancer, it should use the proper DNS name you entered as the Cluster URL.

Click the `Got it` button for the Telemetry consent (or don't, whatever).

Now you can create a new Cluster!
Click the `Add Cluster` button (the big one, in the middle...)
Select `Custom`, enter a name for the cluster in the `Cluster Name` field.
Select `Custom` as the `Cloud Provider` at the bottom, and click Next.

On the next screen, check the `etcd`, `Control Plane` and `Worker` (enabled-by-default) boxes.

After this, copy the given command, and execute it on every node. Theoretically you can do this manually, but who cares.

For each node that joins the cluster, you will get a tiny green notification on the bottom of the page. When you are satisfied, you can click `Done` and you'll be redirected to the Cluster overview screen on which you will see 1 cluster with the Provisioning state. On the top of the screen, you will most likely see the nodes joining, and they will show the steps they are taking during this bootstrapping process.

After a while, when all goes well, the `State` will switch to `Active`, and your cluster will be operational!

# Step 6 - Relaunch Rancher on the cluster.

In the last step we created a simple Rancher installer container which serves as our cluster bootstrapper. But this container itself is not HA of course, so we will restart the Rancher as part of a Kubernetes deployment.

To do this we will need a few dependencies (on your workstation):
- Kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux)
- Helm (https://helm.sh/docs/using_helm/#installing-helm)

In the Rancher browser, select the cluster you just created. On the topright corner click `Kubeconfig File`.
At the bottom there is a `Copy to Clipboard` buttom. Save this file (on your workstation) in the folder ~/.kube/config. This will allow your local kubectl to connect to the cluster.

Once you've done this, verify your kubernetes cluster is working using the following command:

```bash
root@nas:/home/thijs/nuc-cluster# kubectl get nodes
NAME    STATUS   ROLES                      AGE    VERSION
j4105   Ready    controlplane,etcd,worker   3d4h   v1.13.5
nuc1    Ready    controlplane,etcd,worker   3d4h   v1.13.5
nuc2    Ready    controlplane,etcd,worker   3d4h   v1.13.5
```

If you see all nodes, they are all ready and all the roles are configured correctly, you are ready to proceed!

Initialize Helm with the commands:
```bash
kubectl -n kube-system create serviceaccount tiller

kubectl create clusterrolebinding tiller \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:tiller

helm init --service-account tiller
```

This will setup helm on your (rancher) cluster, and gives you access to a whole catalog of automated Kubernetes deployments!

To verify Helm is correctly installed you can use:
```bash
root@nas:/home/thijs/nuc-cluster# helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
```

If the initialization was incorrect, the `Server:` value will be empty.

Add the Rancher repo to Helm using:
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

For SSL support we need cert-manager, this will give Rancher the ability to generate it's own SSL Certificates, and (if you want) even use Let's Encrypt to automatically get signed certificates, but that's beyond the scope of this blog. If you want to use Let's Encrypt, view this page: (https://rancher.com/docs/rancher/v2.x/en/installation/ha/helm-rancher/). For now, we will just use self-signed certificates.

```bash
helm install stable/cert-manager \
  --name cert-manager \
  --namespace kube-system \
  --version v0.5.2
```

Now, because we are going to relaunch Rancher on the cluster, we need to shutdown the old container. If you are living on the edge, you can just delete it. But maybe it's best to just stop it ;)

```bash
thijs@nuc1:~$ sudo docker stop zealous_cartwright
zealous_cartwright
```

(be aware, that the container most definatly has a different name on your system, just look it up using `docker ps`)
Once the container is stopped, we can continue the installation Rancher.

On your workstation:
```bash
helm install rancher-latest/rancher \
  --name rancher \
  --namespace cattle-system \
  --set hostname=k8s.cravu.lan
```
(Replace the hostname, with your hostname of course.)

This will relaunch Rancher on your cluster and it will automatically use a proper setup on the Kubernetes cluster to make it High Available. Once the installation is complete you will see something like:

```bash
root@nas:/home/thijs/nuc-cluster# kubectl get pods -n cattle-system
NAME                                   READY   STATUS    RESTARTS   AGE
cattle-cluster-agent-bb4c6d95b-b2r86   1/1     Running   7          2d5h
cattle-node-agent-crlkk                1/1     Running   7          3d3h
cattle-node-agent-r66vz                1/1     Running   12         3d3h
cattle-node-agent-s2rdt                1/1     Running   12         3d3h
kube-api-auth-55z68                    1/1     Running   6          3d4h
kube-api-auth-htfxf                    1/1     Running   5          3d4h
kube-api-auth-mm2pc                    1/1     Running   6          3d4h
rancher-6fd48d6f59-4dgsb               1/1     Running   6          3d3h
rancher-6fd48d6f59-585gp               1/1     Running   4          2d5h
rancher-6fd48d6f59-xccjq               1/1     Running   6          3d3h
```

If all pods are properly running, you can refresh your browser. You will be prompted again to enter a password for the rancher server. After the initial screens you will be pointed to the Cluster overview.
But instead of an empty list, it should have improted your existing cluster! Cool huh?

Rancher sees it's running on Kubernetes this time, and will automatically import your cluster and make it available to you!

If everything is green on the page, you are set! Happy Ranching!
