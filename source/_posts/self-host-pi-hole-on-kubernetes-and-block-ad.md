---
title: (K3S - 6/8) Self-host Pi-Hole on Kubernetes and block ads and trackers at the network level
date: 2020-04-13 00:00:06
updated: 2024-01-15 00:00:00
---

![](https://gateway.pinata.cloud/ipfs/QmbJb9BztkrFe8ejvxSjkYxwLH79oA7388GEHLQPbqkwnC)

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

Pi Hole is a network-wide ad blocker. In a typical home environment, this can cut out almost all ads to all devices in your home, without having to install an ad blocker on every single device. Technically Pi-Hole acts as a DNS sinkhole which filters out unwanted results using a blacklist domains list. Pi-Hole also offers a great admin interface to configure and analyse your network traffic (DNS, DHCP, Black/White list, regex, etc.).

![](https://gateway.pinata.cloud/ipfs/QmWNdkLpvCo2aHjAL3k45vndybyv6u1aM5wTZw9UgeXZsB)

In this new article, we will learn how to deploy Pi-Hole on a Kubernetes self-hosting platform.



### Prerequisite

In order to run entirely the tutorial, we will need:

- A running Kubernetes cluster (see previous articles if you haven't set this up yet)
- Access to your router admin console to configure Pi-Hole as DNS or disable DHCP (replaced by Pi-Hole)




### Namespace

We are going to isolate all the Kubernetes objects related to Pi-Hole in the namespace `pihole`.

To create a namespace, run the following command:

```
$ kubectl create namespace pihole
```



### Persistence

The first step consists in setting up a volume to store the Pi-Hole config files and data. If you followed the previous articles to install and configure a self-hosting platform using RaspberryPi and Kubernetes, you remember we have on each worker a NFS client pointing to a SSD on `/mnt/ssd`.

**1. Deploy the Persistent Volume (PV)**

The Persistent Volume specify the name, the size, the location and the access modes of the volume:

- The name of the PV is  `pihole`
- The size allocated is 500MB
- The location is `/mnt/ssd/pihole`
- The access is ReadWriteOnce

Create the following file and apply it to the k8 cluster.

```yaml
## pihole.persistentvolume.yml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "pihole"
  labels:
    type: "local"
spec:
  storageClassName: "manual"
  capacity:
    storage: "500Mi"
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/ssd/pihole"
---
```

```
$ kubectl apply -f pihole.persistentvolume.yml
persistentvolume/pihole created
```

You can verify the PV exists with the following command:

```
$ kubectl get pv

NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pihole          500Mi      RWO            Retain           Available           manual                  34s
```


**2. Create the Persistent Volume Claim (PVC)**

The Persistent Volume Claim is used to map a Persistent Volume to a deployment or stateful set. Unlike the PV, the PVC belongs to a namespace.

Create the following file and apply it to the k8 cluster.

```yaml
## pihole.persistentvolumeclaim.yml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "pihole"
  name: "pihole"
spec:
  storageClassName: "manual"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "500Mi"
---
```

```
$ kubectl apply -f pihole.persistentvolumeclaim.yml
persistentvolumeclaim/pihole created
```

You can verify the PVC exists with the following command:

```
$ kubectl get pvc -n pihole

NAME            STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pihole          Bound    pihole          500Mi      RWO            manual         26s
```



### Deployment

In the next part, we are now going to deploy Pi-Hole using a modified version open-source Helm chart [pihole-kubernetes](https://github.com/MoJo2600/pihole-kubernetes).

**1. Install the repo**

```
$ helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/ \ 
  && helm repo update
```


**2. Create a secret to store Pi-Hole admin password**

Replace `<ADMIN_PASSWORD>` by the password of your choice. This password will be used to connect to the Pi-Hole administration interface.

```
$ kubectl create secret generic pihole-secret \
    --from-literal='password=<ADMIN_PASSWORD>' \
    --namespace pihole
```


**3. Download the Chart values of the chart locally**

Run the following command to download the Chart values into the local file `pihole.values.yml`.

```
$ helm show values mojo2600/pihole >> pihole.values.yml
```

If you open the file, you will see the default configuration values to setup Pi-Hole. Instead of using the flag `--set property=value` like before, we will use the file `pihole.values.yml` to make all the changes.


**4. Update the values**

We now need to update a few properties before installing the Helm chart. Open the file `pihole.values.yml` and change the following properties (_replace the information surrounded by <brackets> with your information_).

```yaml
## pihole.values.yml

(...)

persistentVolumeClaim:
  # set to true to use pvc
  enabled: true # Change to true
  # set to true to use you own pvc
  existingClaim: "pihole" # Name of the persistent volume claim

(...)

extraEnvVars:
  TZ: "Europe/London" # Timezone
  ServerIP: 192.168.0.22 # Add the master node IP only if your configure `hostNetwork: true` (next paragraph, point b only)

(...)

## Use an existing secret for the admin password.
admin:
  existingSecret: "pihole-secret" # Reference to the secret created step 2
  passwordKey: "password"
```


The network config might be different if your need to get DHCP working with Pi-hole.

a. I can override my router default DNS config

```yaml
## pihole.values.yml

(...)

serviceTCP:
  type: LoadBalancer # Configure MetalLB to used a dedicated "virtual" IP to expose the DNS server
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc

serviceUDP:
  type: LoadBalancer # Configure MetalLB to used a dedicated "virtual" IP to expose the DNS server
  annotations:
    metallb.universe.tf/allow-shared-ip: pihole-svc
```

b. I can't override my router DNS config and I need to enable DHCP on Pi-Hole (disable on my router) to force the devices to use Pi-Hole as DNS.

In this case, we are going to use the flag `hostNetwork=true` and `privileged=true` to let the pod use the node network with root privilege which will enable DHCP through network broadcast on port 67 ([more info](https://docs.pi-hole.net/docker/DHCP/)).

```yaml
## pihole.values.yml

(...)

serviceTCP:
  type: ClusterIP

serviceUDP:
  type: ClusterIP

(...)

hostNetwork: "true" # The pod uses the host network rather than k8s network (to perform a network broadcast on port 67 / DHCP) See  https://docs.pi-hole.net/docker/DHCP/#docker-pi-hole-with-host-networking-mode
privileged: "true" # Give root permission to the pod on the host
webHttp: "55080" # Random HTTP port to prevent clash (because the pod will be on the host network)
webHttps: "55443" # Random HTTPS port to prevent clash (because the pod will be on the host network)
```


**4. Install the Chart**

In the part, we will install the Helm chart under the namespace `pihole` with `pihole.values.yml` as configuration file.

```
$ helm install pihole mojo2600/pihole \
  --namespace pihole \
  --values pihole.values.yml
```

After a couple of minutes, check if the pod and service is up and running:

```
$ kubectl get pods -n pihole -o wide

NAME                      READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
pihole-695f4bd7c8-xwswq   1/1     Running   1          45m   192.168.0.22   kube-master   <none>           <none>
```

```
$ kubectl get services -n pihole -o wide

NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                 AGE   SELECTOR
pihole        pihole-web                ClusterIP      10.43.148.13    <none>          80/TCP,443/TCP               9s
pihole        pihole-dns-tcp            LoadBalancer   10.43.179.115   192.168.1.241   53:31076/TCP                 9s
pihole        pihole-dns-udp            LoadBalancer   10.43.251.46    192.168.1.241   53:32176/UDP                 9s
pihole        pihole-dhcp               LoadBalancer   10.43.35.57     192.168.1.241   67:31310/UDP                 9s
```




### Ingress

To access Pi-Hole admin, we are now going to deploy an Ingress, responsible of making accessible a service from outside the cluster by mapping an internal `service:port` to a host. To choose a host, we need to configure a DNS like we did for NextCloud "nextcloud.<domain.com>" in the previous article. However, unlike NextCloud, Pi-Hole have no reason to be exposed on the Internet, we can pick a host that will be resolved internally to our Nginx proxy (available at `192.168.0.240` : LoadBalancer IP). The simplest solution is to use [nip.io](https://nip.io) which allows us to map an IP (in our case `192.168.0.240`) to a hostname without touching `/etc/hosts` or configuring a DNS. Basically it resolves `<anything>.<ip>.nip.io` by `<ip>` without requiring anything else, Magic !

**1. Create the file `pihole.ingress.yml`**

Create the following Ingress config file `pihole.ingress.yml` to map the route `/` to Pi-Hole HTTP service:

```yaml
## pihole.ingress.yml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: pihole
  name: pihole-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    cert-manager.io/cluster-issuer: self-signed-issuer
spec:
  tls:
  - hosts:
    - pihole.192.168.1.240.nip.io
    secretName: pihole-tls
  rules:
  - host: pihole.192.168.1.240.nip.io
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: pihole-web
              port:
                number: 80       
---
```


**2. Deploy the ingress**

Deploy the Ingress by applying the file `pihole.ingress.yml`.

```
$ kubectl apply -f pihole.ingress.yml
ingress.extensions/pihole-ingress created
```



### Result

You can now access Pi-Hole via [pihole.192.168.1.190.nip.io/admin](http://pihole.192.168.1.190.nip.io/admin).

![](https://i.imgur.com/FlgpGwV.png)

Click on **Login** on the left menu and enter the password configured earlier in `pihole.values.yml`.

If you configured Pi-Hole to be used as DHCP server, you need to go to **Settings / DHCP**, enable DHCP, configure the IP range and the IP of your network router.

![](https://i.imgur.com/O4BKu8a.png)
