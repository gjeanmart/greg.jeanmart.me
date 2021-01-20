---
title: (K3S - 4/8) Deploy NextCloud on Kuberbetes  The self-hosted Dropbox
date: 2020-04-13 00:00:04
---

![](https://ipfs.infura.io/ipfs/QmWWVPyzfpSc2Pd1EcJ4RyqvQGB6iB7ytNTmVfawExJ3sr)

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

Now we have prepared our RaspberryPi cluster to receive Kubernetes as a self-hosting platform, it's time to start installing applications !

[NextCloud](https://nextcloud.com/) is an file hosting open-source software similar to Dropbox. Unlike Dropbox, NextCloud is not available as a SaaS but only on-premise which means anyone is allowed to install and operate it on their own private server. NextCloud offers individuals and organisations to gain control over their private data with a safe and secure solutions. NextCloud also provides a long list of add-ons working alongside the file sharing solution such as: Calendar & Contacts management, Audio/Video conferencing, Task Management, Photos albums and more.  

![](https://ipfs.infura.io/ipfs/QmQ7zPeatKZ8PY3Y2Qh4USKg9DHEdTiKbKTFjsrPfqrt8T)

In this article we will learn how to safely install NextCloud on a Kubernetes environment and configure both Desktop and Mobile access from anywhere.



### Prerequisite

In order to run entirely the tutorial, we will need:

- A running Kubernetes cluster (see previous articles if you haven't set this up yet)
- A domain name in order to access our NextCloud instance from outside our network. (replace <domain.com> by your domain)
- Have a external static IP (usually the case by default)
- Access to your router admin console to port-forward an incoming request to our Kubernetes Ingress service.




### Namespace

We are going to isolate all the Kubernetes objects related to NextCloud in the namespace `nextcloud`.

To create a namespace, run the following command:

```
$ kubectl create namespace nextcloud
```



### Persistence

The first step consists in setting up a volume to store our NextCloud data (files and database). If you followed the previous articles to install and configure a self-hosting platform using RaspberryPi and Kubernetes, you remember we have on each worker a NFS client pointing to a SSD on `/mnt/ssd`.

**1. Deploy the Persistent Volume (PV)**

The Persistent Volume specify the name, the size, the location and the access modes of the volume:

- The name of the PV is  `nextcloud-ssd`
- The size allocated is 50GB
- The location is `/mnt/ssd/nextcloud`
- The access is ReadWriteOnce

Create the following file and apply it to the k8 cluster.

```yaml
## nextcloud.persistentvolume.yml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "nextcloud-ssd"
  labels:
    type: "local"
spec:
  storageClassName: "manual"
  capacity:
    storage: "50Gi"
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/ssd/nextcloud"
---
```

```
$ kubectl apply -f nextcloud.persistentvolume.yml
persistentvolume/nextcloud-ssd created
```

You can verify the PV exists with the following command:

```
$ kubectl get pv

NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nextcloud-ssd   50Gi       RWO            Retain           Available           manual                  34s
```


**2. Create the Persistent Volume Claim (PVC)**

The Persistent Volume Claim is used to map a Persistent Volume to a deployment or stateful set. Unlike the PV, the PVC belongs to a namespace.

Create the following file and apply it to the k8 cluster.

```yaml
## nextcloud.persistentvolumeclaim.yml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "nextcloud"
  name: "nextcloud-ssd"
spec:
  storageClassName: "manual"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "50Gi"
---
```

```
$ kubectl apply -f nextcloud.persistentvolumeclaim.yml
persistentvolumeclaim/nextcloud-ssd created
```

You can verify the PVC exists with the following command:

```
$ kubectl get pvc -n nextcloud

NAME            STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nextcloud-ssd   Bound    nextcloud-ssd   50Gi       RWO            manual         26s
```



### Deployment

In the next part, we are now going to deploy NextCloud using the [stable/nextcloud Helm chart](https://github.com/helm/charts/tree/master/stable/nextcloud).

**1. Download the Chart values of the chart locally**

Run the following command to download the Chart values into the local file `nextcloud.values.yml`.

```
$ helm show values stable/nextcloud >> nextcloud.values.yml
```

If you open the file, you will see the default configuration values to setup NextCloud. Instead of using the flag `--set property=value` like before, we will use the file `nextcloud.values.yml` to make all the changes.

**2. Update the values**

We now need to update a few properties before installing the Helm chart. Open the file `nextcloud.values.yml` and change the following properties (_replace the information surrounded by <brackets> with your information_).

```yaml
## nextcloud.values.yml
nextcloud:
  host: "nextcloud.<domain.com>" # Host to reach NextCloud
  username: "admin" # Admin
  password: "<PASSWORD>" # Admin Password
(...)
persistence:
  enabled: true # Change to true
  existingClaim: "nextcloud-ssd" # Persistent Volume Claim created earlier
  accessMode: ReadWriteOnce
  size: "50Gi"
```

Take a look at the file if you want to make more customisation to NextCloud:

- Configure emails
- Configure an external database or deploy a MariaDB as part of the Chart (improve performance)
- Configure Redis server (improve performance)


**3. Install the Chart**

In the part, we will install the Helm chart under the namespace `nextcloud` with `nextcloud.values.yml` as configuration file.

```
$ helm install nextcloud stable/nextcloud \
  --namespace nextcloud \
  --values nextcloud.values.yml
```

After a couple of minutes, check if the pod and service is up and running:

```
$ kubectl get pods -n nextcloud

NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE          NOMINATED NODE   READINESS GATES
nextcloud-78f5564f89-854jr   1/1     Running   0          15m   10.42.0.16   kube-master   <none>           <none>
```

```
$ kubectl get services -n nextcloud -o wide

NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
nextcloud   ClusterIP   10.43.188.121   <none>        8080/TCP   51m   app.kubernetes.io/name=nextcloud
```

**4. Debugging**

You can check the logs with the following command:

```
$ kubectl logs -f nextcloud-78f5564f89-854jr -n nextcloud
```

Otherwise check the folder `/mnt/ssd/nextcloud/data/nextcloud.log`.



### Outside access

_This step is configured before the ingress in order to be able to issue a certificate automatically when we deploy the ingress_

The next part consist to enable the connections to NextCloud from outside so you can access your data from anywhere.

**1. Port Forwarding**

First you need to go to your router setup and add a port-forwarding rule to map any incoming requests on port 80 or port 443 to be forwarded to `192.168.0.240` (the LoadBalancer IP of the Nginx).

![](https://i.imgur.com/iqpI0H3.png)

_VirginHub - Port-Forwarding_


**2. Map the subdomain `nextcloud.<domain.com>` to your home router**

First you need to find out what's your router external IP, run this command or go to [whatismyip.com](https://www.whatismyip.com).

```
$ curl ipecho.net/plain
x.x.x.x
```

Then, we need to configure our subdomain to make sure `nextcloud.<domain.com>` resolves to our external static IP. Go to your domain provider console / DNS management add a record:

- **Type:** A
- **Name:** nextcloud (subdomain)
- **Value:** x.x.x.x (external satic IP)

![](https://i.imgur.com/b0BKzIE.png)

_GoDaddy_



### Ingress

At this point, the application (pod `nextcloud-78f5564f89-854jr`) is only accessible within the cluster on port 8080. To make it accessible from outside the cluster (on our network), we need to deploy an Ingress mapping **service:port** to a route of the Nginx proxy.

Because, this route will also be exposed over the Internet, we will also issue a certificate to encrypt the traffic with SSL.

**1. Create the ingress config file**

Create the file `nextcloud.ingress.yml` containing:

```yaml
## nextcloud.ingress.yml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: "nextcloud" # Same namespace as the deployment
  name: "nextcloud-ingress" # Name of the ingress (see kubectl get ingress -A)
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod" # Encrypt using the ClusterIssuer deployed while setting up Cert-Manager
    nginx.ingress.kubernetes.io/proxy-body-size:  "50m" # Increase the size of the maximum allowed size of the client request body
spec:
  tls:
  - hosts:
    - "nextcloud.<domain.com>" # Host to access nextcloud
    secretName: "nextcloud-prod-tls" # Name of the certifciate (see kubectl get certificate -A)
  rules:
  - host: "nextcloud.<domain.com>" # Host to access nextcloud
    http:
      paths:
        - path: /  # We will access NextCloud via the URL https://nextcloud.<domain.com>/
          backend:
            serviceName: "nextcloud" # Mapping to the service (see kubectl get services -n nextcloud)
            servicePort: 8080 # Mapping to the port (see kubectl get services -n nextcloud)
---
```

**2. Deploy the Ingress**

Now deploy the ingress:

```
$ kubectl apply -f nextcloud.ingress.yml
ingress.extensions/nextcloud-ingress created
```

**3. Check the certificate issuance**

After you deployed the ingress, a certificate should be issued, check the `certificaterequest` and `certificate` (it might take a couple of minutes to be READY):

```
$ kubectl get certificaterequest -n nextcloud -o wide

NAMESPACE   NAME                         READY   ISSUER             STATUS                                         AGE
nextcloud   nextcloud-prod-tls-9929727   True    letsencrypt-prod   Certificate fetched from issuer successfully   17m


$ kubectl get certificate -n nextcloud -o wide

NAMESPACE   NAME                 READY   SECRET                           ISSUER             STATUS                                          AGE
nextcloud   nextcloud-prod-tls   True    nextcloud-prod-tls   letsencrypt-prod   Certificate is up to date and has not expired   17m
```



### Conclusion

Alright, still with me :) You can now try to access your NextCloud instance using your browser, mobile or the Android/iOS app from home or outside via [https://nextcloud.<domain.com>](https://nextcloud.<domain.com>).

![](https://i.imgur.com/HjjUi7I.png)

Connect with the user admin and the password configured in the file `nextcloud.values.yml`.

![](https://i.imgur.com/JtD053z.png)

You can also download the [Android](https://play.google.com/store/apps/details?id=com.nextcloud.client&hl=en_GB) or [iOS](https://apps.apple.com/gb/app/nextcloud/id1125420102) app and access you data, sync automatically your photos and more.

![](https://i.imgur.com/ckk4dEl.png)
