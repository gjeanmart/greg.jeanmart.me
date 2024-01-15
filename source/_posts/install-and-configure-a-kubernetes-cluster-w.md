---
title: (K3S - 3/8) Install and configure a Kubernetes cluster with k3s to self-host applications
date: 2020-04-13 00:00:03
updated: 2024-01-15 00:00:00
---

![](https://gateway.pinata.cloud/ipfs/QmNij6VSLpX25UNBJaPm4FowsaLMqPE5yyRQh92Zv3GmpZ)

This article is part of the series *Build your very own self-hosting platform with Raspberry Pi and Kubernetes*

1. [Introduction](/2020/04/13/build-your-very-own-self-hosting-platform-wi)
2. [Install Raspbian Operating-System and prepare the system for Kubernetes](/2020/04/13/install-raspbian-operating-system-and-prepar)
3. [Install and configure a Kubernetes cluster with k3s to self-host applications](/2020/04/13/install-and-configure-a-kubernetes-cluster-w)
4. [Deploy NextCloud on Kuberbetes: The self-hosted Dropbox](/2020/04/13/deploy-nextcloud-on-kuberbetes--the-self-hos)
5. [Self-host your Media Center On Kubernetes with Plex, Sonarr, Radarr, Transmission and Jackett](/2020/04/13/self-host-your-media-center-on-kubernetes-wi)
6. [Self-host Pi-Hole on Kubernetes and block ads and trackers at the network level](/2020/04/13/self-host-pi-hole-on-kubernetes-and-block-ad)
7. [Self-host your password manager with Bitwarden](/2020/04/13/self-host-your-password-manager-with-bitward)
8. [Deploy Prometheus and Grafana to monitor a Kubernetes cluster](/2020/04/13/deploy-prometheus-and-grafana-to-monitor-a-k)

### Introduction

In the previous article, we freshly prepared three machines (one master and two workers). In this article, we are going to learn how to install Kubernetes using [k3s](https://k3s.io), a lightweight version of Kubernetes, suitable for ARM-based computers such as Raspberry Pi. If you need any support with k3s, I recommend checking the [official documentation](https://rancher.com/docs/k3s/latest/en/) as well as the [GitHub repository](https://github.com/rancher/k3s).

Once the cluster is up and each node connected to each other, we will install some useful services such as:

- **[Helm](https://helm.sh/):** Package manager for Kubernetes
- **[MetalLB](https://metallb.universe.tf/):** Load-balancer implementation for bare metal Kubernetes clusters
- **[Cert Manager](https://cert-manager.io):** Native Kubernetes certificate management controller.
- **[Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/):** A web-based Kubernetes user interface




### Install k3s server (master node)

In the first part of this article, we will install the Kubernetes master node which represents the orchestrator of the cluster.

**1. Connect via ssh to the master node**

```
$ ssh pi@192.168.0.22
```


**2. Configure the following environment variables**

The first line specifies in which mode we would like to write the k3s configuration (required when not running commands as `root`) and the second line actually says k3s not to deploy its default load balancer named _servicelb_, instead we will install manually [_metalb_](https://metallb.universe.tf/) which is in my opinion better and more widely used.

```
$ export K3S_KUBECONFIG_MODE="644"
$ export INSTALL_K3S_EXEC=" --disable=servicelb"
```


**3. Run the installer**

The next command simply downloads and executes the k3s installer. The installation will take into account the environment variables set just before.

```
$ curl -sfL https://get.k3s.io | sh -

[INFO]  Finding release for channel stable
[INFO]  Using v1.28.4+k3s2 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.28.4+k3s2/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.28.4+k3s2/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  Host iptables-save/iptables-restore tools not found
[INFO]  Host ip6tables-save/ip6tables-restore tools not found
[INFO]  systemd: Starting k3s
```


**4. Verify the status**

The installer creates a systemd service which can be used for `stop`, `start`, `restart` and verify the `status` of the k3s server running Kubernetes.

```
$ sudo systemctl status k3s

● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; preset: enabled)
     Active: active (running) since Sun 2023-12-31 13:34:57 GMT; 21s ago
       Docs: https://k3s.io
    Process: 1695 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service (code=exited, status=0/SUCCESS)
    Process: 1697 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
    Process: 1698 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 1699 (k3s-server)
      Tasks: 57
     Memory: 484.3M
        CPU: 1min 45.687s
     CGroup: /system.slice/k3s.service
             ├─1699 "/usr/local/bin/k3s server"
             └─1804 "containerd "
(...)
```

k3s also installed the [Kubernetes Command Line Tools](https://kubernetes.io/docs/reference/kubectl/overview/) `kubectl`, so it is possible to start querying the cluster (composed at this stage, of only one node - the master, and a few internal services used by Kubernetes).

- To get the details of the nodes

```
$ kubectl get nodes -o wide

NAME          STATUS   ROLES                  AGE   VERSION        INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION      CONTAINER-RUNTIME
kube-master   Ready    control-plane,master   51s   v1.28.4+k3s2   192.168.1.20   <none>        Debian GNU/Linux 12 (bookworm)   6.1.0-rpi7-rpi-v8   containerd://1.7.7-k3s1
```

- To get the details of all the services deployed

```
$ kubectl get pods -A -o wide

NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE   IP          NODE          NOMINATED NODE   READINESS GATES
kube-system   local-path-provisioner-84db5d44d9-jx6sj   1/1     Running   0          42s   10.42.0.4   kube-master   <none>           <none>
kube-system   coredns-6799fbcd5-ztjxv                   1/1     Running   0          42s   10.42.0.3   kube-master   <none>           <none>
kube-system   metrics-server-67c658944b-5dlg2           0/1     Running   0          42s   10.42.0.2   kube-master   <none>           <none>
```


**5. Save the access token**

Each agent will require an access token to connect to the server, the token can be retrieved with the following commands:

```
$ sudo cat /var/lib/rancher/k3s/server/node-token

K106edce2ad174510a840ff7e49680fc556f8830173773a1ec1a5dc779a83d4e35b::server:5a9b70a1f5bc02a7cf775f97fa912345
```



### Install k3s agent (worker nodes)

In the second part, we are now installing the k3s agent to connect on each worker to the k3s server (master).


**1. Connect via ssh to the worker node**

```
$ ssh pi@192.168.1.21
```


**2. Configure the following environment variables**

The first line specifies in which mode we would like to write the k3s configuration (required when not running command as `root`) and the second line provide the k3s server endpoint the agent needs to connect to. Finally, the third line is an access token to the k3s server saved previously.

```
$ export K3S_KUBECONFIG_MODE="644"
$ export K3S_URL="https://192.168.1.20:6443"
$ export K3S_TOKEN="K106edce2ad174510a840ff7e49680fc556f8830173773a1ec1a5dc779a83d4e35b::server:5a9b70a1f5bc02a7cf775f97fa912345"
```


**3. Run the installer**

The next command simply downloads and executes the k3s installer. The installation will take into account the environment variables set just before and install the agent.

```
$ curl -sfL https://get.k3s.io | sh -

[INFO]  Finding latest release
[INFO]  Using v1.17.0+k3s.1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.17.0+k3s.1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.17.0+k3s.1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent
```


**4. Verify the status**

The installer creates a systemd service which can be used for `stop`, `start`, `restart` and verify the `status` of the k3s agent running Kubernetes.

```
$ sudo systemctl status k3s-agent

● k3s-agent.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s-agent.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-01-12 16:03:07 UTC; 9s ago
     Docs: https://k3s.io
  Process: 4970 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
  Process: 4973 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
 Main PID: 4974 (k3s-agent)
    Tasks: 16
   Memory: 205.1M
(...)
```

k3s also installed the [Kubernetes Command Line Tools](https://kubernetes.io/docs/reference/kubectl/overview/) `kubectl`, so it is possible to start querying the cluster and observe the all nodes are reconciliated.

```
$ kubectl get nodes -o wide

NAME           STATUS   ROLES    AGE     VERSION         INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
kube-master    Ready    master   44h     v1.17.0+k3s.1   192.168.0.22   <none>        Raspbian GNU/Linux 10 (buster)   4.19.75-v7l+       containerd://1.3.0-k3s.5
kube-worker1   Ready    <none>   2m47s   v1.17.0+k3s.1   192.168.0.23   <none>        Debian GNU/Linux 10 (buster)     5.4.6-rockchip64   containerd://1.3.0-k3s.5
kube-worker2   Ready    <none>   2m9s    v1.17.0+k3s.1   192.168.0.24   <none>        Raspbian GNU/Linux 10 (buster)   4.19.75-v7+        containerd://1.3.0-k3s.5
```

**5. Configure the environment variable KUBECONFIG**
In order to be read by other tools (cf. Helm), we need to configure the environment variable `KUBECONFIG` to point to the K3S config

```
$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

_Note: Put this line into your `~/.bashrc` if you want to have this environment variable still up after a restart._

### Connect remotely to the cluster

If you don't want to connect via SSH to a node every time you need to query your cluster, it is possible to install `kubectl` (k8s command line tool) on your local machine and control remotely your cluster.

**1. Install kubectl on your local machine**

Read the [following page](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to know how to install `kubectl` on Linux, MacOS or Windows.


**2. Copy the k3s config file from the master node to your local machine**

The command `scp` allows to transfer file via SSH from/to a remote machine. We simply need to download the file `/etc/rancher/k3s/k3s.yaml` located on the master node to our local machine into `~/.kube/config`.

```
$ scp pi@192.168.1.20:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

The file contains a localhost endpoint `127.0.0.1`, we just need to replace this by the IP address of the master node instead (in my case `192.168.1.20`).

```
$ sed -i '' 's/127\.0\.0\.1/192\.168\.1\.20/g' ~/.kube/config
```


**3. Try using `kubectl` from your local machine**

```
$ kubectl get nodes -o wide

NAME           STATUS   ROLES    AGE   VERSION         INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
kube-worker1   Ready    <none>   18m   v1.17.0+k3s.1   192.168.1.22   <none>        Debian GNU/Linux 10 (buster)     5.4.6-rockchip64   containerd://1.3.0-k3s.5
kube-worker2   Ready    <none>   17m   v1.17.0+k3s.1   192.168.1.21   <none>        Raspbian GNU/Linux 10 (buster)   4.19.75-v7+        containerd://1.3.0-k3s.5
kube-master    Ready    master   44h   v1.17.0+k3s.1   192.168.1.20   <none>        Raspbian GNU/Linux 10 (buster)   4.19.75-v7l+       containerd://1.3.0-k3s.5
```





### Install Helm (version >= 3.x.y) - Kubernetes Package Manager

[Helm](https://helm.sh/) is a package manager for Kubernetes. An application deployed on Kubernetes is usually composed of multiple config files (deployment, service, secret, ingress, etc.) which can be more or less complex and are generally the same for common applications.

Helm provides a solution to define, install, upgrade k8s applications based on config templates (called _charts_). A simple unique config file (names Values.yml) is used to generate all the necessary k8s config files and deploy them. The repository [hub.helm.sh](https://hub.helm.sh/) contains all the "official" charts available but you can easily find unofficial charts online.

**1. Install Helm command line tools on your local machine**

Refer to the [following page](https://helm.sh/docs/intro/install) to install `helm` on your local machine. **You must install Helm version >= 3.x.y**

Example for Linux / MacOS:

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
Downloading https://get.helm.sh/helm-v3.13.1-linux-arm64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm
```

**2. Check the version**

Verify that you have Helm version 3.x installed.

```
$ helm version
version.BuildInfo{Version:"v3.13.1", GitCommit:"3547a4b5bf5edb5478ce352e18858d8a552a4110", GitTreeState:"clean", GoVersion:"go1.20.8"}
```


**3. Add the repository for official charts**

Configure the repository `stable https://kubernetes-charts.storage.googleapis.com` to access the [official charts](https://github.com/helm/charts/tree/master/stable)

```
$ helm repo add stable https://charts.helm.sh/stable
"stable" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```



We can now install application using Helm using `helm install <deployment_name> <chart_name> --namespace <namespace> --set <property_value_to_change>`, uninstall application running `helm uninstall <deployment_name> --namespace <namespace>` and list the applications with `helm list --namespace <namespace>`

I also recommend checking this page to learn more [how to use Helm cli](https://helm.sh/docs/intro/using_helm/).




### Install MetalLB - Kubernetes Load Balancer

[MetalLB](https://metallb.universe.tf) is a load-balancer implementation for bare metal Kubernetes clusters. When configuring a [Kubernetes service](https://kubernetes.io/docs/concepts/services-networking/service/) of type _LoadBalancer_, MetalLB will dedicate a virtual IP from an address-pool to be used as load balancer for an application.

To install MetalLB from Helm, you simply need to run the following command `helm install ...` with:

- `metallb`: the name to give to the deployment
- `stable/metallb`: the name of the [chart](https://metallb.universe.tf/installation/#installation-with-helm)
- `--namespace kube-system`: the namespace in which we want to deploy MetalLB.

```
$ helm repo add metallb https://metallb.github.io/metallb
$ helm repo update

$ helm install metallb metallb/metallb --namespace kube-system 
```

After a few seconds, you should observe the MetalLB components deployed under `kube-system` namespace.

```
$ kubectl get pods -n kube-system -l app.kubernetes.io/name=metallb -o wide

NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
kube-system   metallb-speaker-s7cvp                     1/1     Running   0          2m47s   192.168.1.20   kube-master    <none>           <none>
kube-system   metallb-speaker-jx64v                     1/1     Running   0          2m47s   192.168.1.21   kube-worker1   <none>           <none>
kube-system   metallb-controller-6fb88ff94b-4g256       1/1     Running   0          2m47s   10.42.1.7      kube-worker1   <none>           <none>
kube-system   metallb-speaker-k5kbh                     1/1     Running   0          2m47s   192.168.1.22   kube-worker2   <none>           <none>
```

Now, we need to configure the Layer-2 settings in MetalLB, including the IP addresses reserved for the load balancer.

```
$ kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: k3s-lb-pool
  namespace: kube-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.249
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: k3s-lb-pool
  namespace: kube-system
spec:
  ipAddressPools:
  - k3s-lb-pool
EOF
```


All done. No every time a new Kubenertes service of type _LoadBalancer_ is deployed, MetalLB will assign an IP from the pool to access the application.


### Install Nginx - Web Proxy (DEPRECATED)

[Nginx](https://www.nginx.com/) is a recognized high-performance web server / reverse proxy. It can be used as [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) to expose HTTP and HTTPS routes from outside the cluster to services within the cluster.

Similarly to MetalLB, we will use the following [stable/nginx-ingress Helm chart](https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/installation-with-helm/) to install our proxy server.


```
$ helm repo add nginx-stable https://helm.nginx.com/stable
$ helm repo update


$ helm install nginx-ingress nginx-stable/nginx-ingress --namespace kube-system
```


After a few seconds, you should observe the Nginx component deployed under `kube-system` namespace.

```
$ kubectl get pods -n kube-system -l app.kubernetes.io/name=nginx-ingress -o wide

NAME                                             READY   STATUS    RESTARTS   AGE     IP          NODE           NOMINATED NODE   READINESS GATES
nginx-ingress-controller-996c5bf9-k4j64   1/1     Running   0          76s   10.42.1.13   kube-worker1   <none>           <none>
```

Interestingly, Nginx service is deployed in LoadBalancer mode, you can observe MetalLB allocates a virtual IP (column ` EXTERNAL-IP`) to Nginx with the command here:

```
$ kubectl get services  -n kube-system -l app.kubernetes.io/name=nginx-ingress -o wide

NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
nginx-ingress-controller   LoadBalancer   10.43.253.188   192.168.1.190   80:30423/TCP,443:32242/TCP   92s   app=nginx-ingress,component=controller,release=nginx-ingress
```

From you local machine, you can try to externally access Nginx via the LoadBalancer IP (in my case `http://192.168.1.190`) and it should return the following message "404 not found" because nothing is deployed yet.

![](https://i.imgur.com/DQI2Nwr.png)




### Install cert-manager

[Cert Manager](https://cert-manager.io) is a set of Kubernetes tools used to automatically deliver and manage x509 certificates against the ingress (Nginx in our case) and consequently secure via SSL all the HTTP routes with almost no configuration.

**1. Install the CustomResourceDefinition**

Install the CustomResourceDefinition resources.

```
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml
```


**2. Configure the jetstack Helm repository**

cert-manager Helm charts aren't hosted by the offical Helm hub, you need to configure a new repository named JetStack which maintains those charts ([here](https://github.com/jetstack/cert-manager/tree/master/deploy/charts/cert-manager)).

```
$ helm repo add jetstack https://charts.jetstack.io \
  && helm repo update
```


**3. Install cert-manager through Helm**

Run the following command to install the cert-manager components under the `kube-system` namespace.

```
$ helm install cert-manager jetstack/cert-manager \
  --namespace kube-system \
  --version v1.13.3
```

Check that all three cert-manager components are running.

```
$ kubectl get pods -n kube-system -l app.kubernetes.io/instance=cert-manager -o wide

NAME                                       READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
cert-manager-cainjector-6659d6844d-w9vrn   1/1     Running   0          69s   10.42.1.13   kube-worker1   <none>           <none>
cert-manager-859957bd4c-2nzqp              1/1     Running   0          69s   10.42.0.16   kube-master    <none>           <none>
cert-manager-webhook-547567b88f-zfm9x      1/1     Running   0          68s   10.42.0.17   kube-master    <none>           <none>
```


**4. Configure the certificate issuers**

We now going to configure two certificate issuers from which signed x509 certificates can be obtained, such as [Let’s Encrypt](https://letsencrypt.org):

- _letsencrypt-staging_: will be used for testing purpose only
- _letsencrypt-prod_: will be used for production purpose.

Run the following commands (change `<YOUR EMAIL>` by your email).

```
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: <YOUR EMAIL>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          ingressClassName: traefik          
EOF
```

```
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <YOUR EMAIL>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          ingressClassName: traefik             
EOF
```

We can also add a ClusterIssuer for self-signed certficates for anything running locally (within our private network)
```
$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: self-signed-issuer
spec:
  selfSigned: {}
EOF
```

Once done, we should be able to automatically issue a Let's Encrypt's certificate every time we configure an ingress with ssl.


**5. Example: Configuration of a ingress with SSL**

The following k8s config file allows to access the service `service_name` (port `80`) from outside the cluster with issuance of a certificate to the domain `domain`.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    cert-manager.io/cluster-issuer: self-signed-issuer | letsencrypt-staging | letsencrypt-prod
spec:
  tls:
  - hosts:
    - <domain>
    secretName: "<domain>-tls"
  rules:
  - host: <domain>
    http:
      paths:
        - path: /
          backend:
            serviceName: <service_name>
            servicePort: 80
---
```             



### Manage storage

Most of the application require a persistence storage to store data and allow running containers to access this storage space from any nodes of the cluster. In the previous chapter, we set up a Persistent Volume to access the SSD connected to the master node via NFS.

**1. Deploy the Persistent Volume**

In order to expose a NFS share to our applications deployed on Kubernetes, we will need first to define a Persistent Volume. Apply the following config:

```yaml
## example.nfs.persistentvolume.yml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-ssd-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/ssd/example"
---
```

```
$ kubectl apply -f example.nfs.persistentvolume.yml
```


**2. Deploy the Persistent Volume Claim**

Now, we need to configure a Persistent Volume Claim which maps a Peristent Volume to a Deployment or Statefulset. Apply the

```yaml
## example.nfs.persistentvolumeclaim.yml
---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: example-ssd-volume
  spec:
    storageClassName: manual
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
---
```

```
$ kubectl apply -f example.nfs.persistentvolumeclaim.yml
```



**3. Checkout the result**

You should be able to query the cluster to find our Persistent Volume and Persistent Volume Claim.

```
$ kubectl get pv

NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
example-ssd-volume   1Gi        RWO            Retain           Available           manual                  3s

$ kubectl get pvc -A

NAMESPACE   NAME                 STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   AGE
default     example-ssd-volume   Bound    example-ssd-volume   1Gi        RWO            manual         6s
```

This method will be used to declare persistent storage volume for each of our application. You can learn more about Persistent Volume [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).




### Install Kubernetes Dashboard

[Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) is a web-based Kubernetes user interface allowing similar operations as _kubectl_.

![](https://kubernetes.io/images/docs/ui-dashboard.png)



**1. Install kubernetes-dashboard via Helm**


```
$ helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
$ helm repo update
$ helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --namespace kube-system
```


After a few seconds, you should see to pods running in the namespace `kube-system`.

```
$ kubectl get pods -n kube-system -l app.kubernetes.io/name=kubernetes-dashboard

NAME                                        READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-65cd84fc57-gspfp       1/1     Running   0          26m
```


**2. Create `admin-user` to connect kubernetes-dashboard**

According to the [wiki](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md), create a new user named `admin-user` to grant this user admin permissions and login to Dashboard using bearer token tied to this user.

```
$ cat <<EOF | kubectl apply -f -
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
---
EOF
```


**3. Retrieve the unique access token of `admin-user`**

In order to authenticate to the dashboard web page, we will need to provide a token we can retrieve by executing the command:

```
$ kubectl -n kube-system create token admin-user
GscCR9APmOxm53jwLj8XFqw [COPY HERE]
```

*Copy the token value*


**4. Create a secure channel to access the kubernetes-dashboard**

To access kubernetes-dashboard from your local machine, you must create a secure channel to your Kubernetes cluster. Run the following command:

```
$  kubectl proxy

Starting to serve on 127.0.0.1:8001
```


**5. Connect to kubernetes-dashboard**

Now we have a secure channel, you can access kubernetes-dashboard via the following URL: [http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:https/proxy/](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:https/proxy/).

Select "Token", copy/paste the token previously retrieved and click on "Sign in".

![](https://i.imgur.com/7xW77VN.png)

Well done, you have now access to a nice web interface to visialise and manage all the Objects of your Kubernetes cluster (*you can switch namespace with the dropdown on the left menu*).

![](https://i.imgur.com/Ky9zXpH.png)



### Conclusion

In conclusion of this article, we now have a ready to use Kubernetes cluster to self-host applications. In the next articles, we will learn how to deploy specific applications such as a Media manager with Plex, a self-hosted file sharing solution similar to DropBox and more.




### Teardown

If you want to uninstall completely the Kubernetes from a machine.

**1. Worker node**

Connect to the worker node and run the following commands:

```
$ /usr/local/bin/k3s-agent-uninstall.sh
```


**2. Master node**

Connect to the master node and run the following commands:

```
$ /usr/local/bin/k3s-uninstall.sh
```




### Known Issues

- cert-manager doesn't issue a certificate, it could be a DNS problem: [Cert Manager works! (Jim Nicholson)](https://project.kube.thejimnicholson.com/2020/04/17/cert-manager-works.html)
