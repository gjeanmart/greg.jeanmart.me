---
title: (K3S - 2/8) Install Raspbian Operating-System and prepare the system for Kubernetes
date: 2020-04-13 00:00:02
updated: 2024-01-15 00:00:00
---

![](https://gateway.pinata.cloud/ipfs/Qma6gsqaPkdAEKDzke9N7DTHXKKGRnU2RD7sH7tXdLSUjt)

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

Fist of all, we need to install and configure **Raspbian Linux Operating System** on each node of the future Kubernetes cluster.

Our cluster will be composed of three machines (I might use the terms _device_, _machine_, _node_ or _host_, that's all the same! a Single-Board Computer used as part of our future cluster):

| Hostname | IP | Description |
|---|---|---|
| kube-master | 192.168.1.20 | A Master represents the main node of the cluster responsible of the orchestration. It can act as a worker as well and run applications |
| kube-worker1 | 192.168.1.21 | A Worker is a machine dedicated to run applications only. It is remotely managed by the master node |
| kube-worker2 | 192.168.1.22 | A Worker is a machine dedicated to run applications only. It is remotely managed by the master node |


We are using a Portable SSD connected to the master node and exposed to the worker via NFS to store the volume data.

![](https://gateway.pinata.cloud/ipfs/QmfAnBjKB9hj2CMikh4c5TVKHLoKABBS5B5yQbSRRXyAtZ)



**Notes:**

- _Do not forget to run each step on each node (unless specified)_
- _We assume here your local network is under 192.168.0.x. You might need to change to match your home network._
- _If you prefer to use a NAS rather than a SSD, skip the part "Configure the SSD disk share" but configure the NFS client on each machine._
- _When I refer to the "local machine", it's usually your laptop or Desktop PC from where you are reading this._



### Flash the OS on the Micro SD card


**1. Download Raspberry Pi Imager**

Go to the [download page](https://www.raspberrypi.com/software/) and download Raspberry Pi Imager depending on your OS.

![](https://i.ibb.co/X7rWSbT/Screenshot-at-Dec-31-13-55-46.png)


**2. Plug an Micro SD Card into your local machine**

**3. Launch Raspberry Pi Imager**

- Choose your Raspberry Pi device version, Operating System (I recommend the LITE version) and the location of your SD card. Press "Next"

![](https://i.ibb.co/R08VPs0/Screenshot-at-Dec-31-13-57-05.png)


- Click on "Edit Settings"
You can set the hostname (`kube-master` for instance), the SSH username/password and the WIFI (if your Raspberry PI supports it). Press "SAVE"

![](https://i.ibb.co/8xYZtBs/image-2.png)

Press "YES"

![](https://i.ibb.co/VtqFY00/Screenshot-at-Dec-31-14-01-21.png)


Wait for the image to be flashed on the SD card

![](https://i.ibb.co/P1XKJxm/Screenshot-at-Dec-31-14-02-24.png)


**6. Unplug the Micro SD Card from your local machine and plug it to the Raspberry Pi**

**7. Plug the power to the Raspberry Pi as well as an Ethernet cable (only if you haven't configure WIFI)**


### Power up and connect via SSH

After you power up each device, we will attempt to connect from our local machine to the node via SSH. If you are under Linux or MacOS, toy only need to open a new terminal. For Microsoft Windows users, you can download and use [Putty](https://www.putty.org) as SSH client.

**1. Determine the device IP address**

Your network router probably assigns an arbitrary IP address when a device tries to join the network via DHCP. To find the address attributed to the device, you can check either on your router admin panel (usually http://192.168.1.1 assuming your local network is 192.168.1.x) or via a tool like [angryip](https://angryip.org).

![](https://i.ibb.co/gSRwyw6/Screenshot-at-Dec-31-14-04-58.png)

_E.g. Freebox_

In my case, the device named _raspberrypi_ (hostname) is assigned to the IP address **192.168.1.20**.

**2. Connect via SSH to the machine**

From a new terminal (or Putty), execute the following command `ssh pi@<IP>` to connect remotely to the node. You will be asked to accept to establish the connection (answer `yes`) and then to enter the password. The default password after a fresh Raspbian installation is `raspberry` (this will be change after this step).

```
greg@laptop:~$ ssh pi@192.168.1.20
The authenticity of host '192.168.1.20 (192.168.1.20)' can't be established.
ED25519 key fingerprint is SHA256:8EkC38flpX0BGUIuP/fVRI/E2NqX6V1rNZzCXXXXX.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.20' (ED25519) to the list of known hosts.
pi@192.168.1.20's password:
Linux kube-master 6.1.0-rpi7-rpi-v8 #1 SMP PREEMPT Debian 1:6.1.63-1+rpt1 (2023-11-24) aarch64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent permitted by applicable law

pi@kube-master:~ $
```

Well done, you are now connected remotely to the machine. Do the same for the other machines of the cluster.


### Configure the OS

Before starting installing the Kubernetes cluster, we need to run a few common steps and security checks.

#### Upgrade the system

To make sure, the system is up-to-date, run the following command to download the latest update and security patches. This step might take a few minutes.

```
pi@kube-master:~ $ sudo apt-get update && sudo apt-get upgrade -y

Get:1 http://raspbian.raspberrypi.org/raspbian buster InRelease [15.0 kB]
Get:2 http://archive.raspberrypi.org/debian buster InRelease [25.1 kB]
Get:3 http://raspbian.raspberrypi.org/raspbian buster/main armhf Packages [13.0 MB]
Get:4 http://archive.raspberrypi.org/debian buster/main armhf Packages [261 kB]
99% [3 Packages store 0 B]
(...)  
Processing triggers for dbus (1.12.16-1) ...
Processing triggers for install-info (6.5.0.dfsg.1-4+b1) ...
Processing triggers for mime-support (3.62) ...
Processing triggers for libc-bin (2.28-10+rpi1) ...
```


#### Configure a static IP

_Note: This could be also done at the network level via the router admin (DHCP)._

By default, the router assigns a arbitrary IP address to the device which means it is highly possible that the router will assign a new different IP address after a reboot. To avoid to recheck our router, it is possible to assign a static IP to the machine.  

Edit the file `/etc/dhcpcd.conf` and add the four lines below:

```
pi@raspberrypi:~ $ sudo vi /etc/dhcpcd.conf

interface eth0
static ip_address=192.168.1.<X>/24
static routers=192.168.1.1
static domain_name_servers=1.1.1.1
```


_Struggling with `vi`? take a look at the [Vim cheat-sheet](https://devhints.io/vim) or alternatively, use nano._



#### Enable container features

We need to enable _container features_ in the kernel in order to run containers.

Edit the file `/boot/cmdline.txt`:

```
pi@kube-master:~ $ sudo vi /boot/cmdline.txt
```

and add the following properties **at the end of the line**:

```
elevator=deadline cgroup_memory=1 cgroup_enable=memory
```

#### Restart and connect to the static IP with the new password and check the hostname.

```
pi@kube-master:~ $ sudo reboot

Connection to 192.168.1.20 closed by remote host.
Connection to 192.168.1.20 closed.
```

Reconnect after a few seconds

```
greg@laptop:~$ ssh pi@192.168.1.20
pi@192.168.1.20's password: <PASSWORD>
```

Check if the hostname has been updated

```
pi@kube-master:~ $ hostname

kube-master
```


<!--
DEPRECATED
 #### Firewall

Switch Debian firewall to legacy config:

```shell
$ sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
$ sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
``` -->



### Configure the SSD disk share

As explained during the introduction, I made the choice to connect a portable SSD to the Master node and gave access via NFS to each worker.


**====== Master node only - Mount the disk and expose a NFS share ======**



**A. Mount the disk to the master**

**1. Plug the SSD to the USB3.0 (blue) port**

**2. Find the disk name (drive)**

Run the command `fdisk -l` to list all the connected disks to the system (includes the RAM) and try to identify the SSD. The disk which has a size of *465.6 GiB* and a model name *Portable SSD T5* and located into `/dev/sda` is our SSD.

```
pi@kube-master:~ $ sudo fdisk -l

Disk /dev/ram0: 4 MiB, 4194304 bytes, 8192 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
(...)
Disk /dev/sda: 465.8 GiB, 500107862016 bytes, 976773168 sectors
Disk model: Portable SSD T5
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33553920 bytes
Disklabel type: dos
Disk identifier: 0x41d0909f
```


**3. Create a partition**

If your disk is new and freshly out of the package, you will need to create a partition.

```
pi@kube-master:~ $ sudo mkfs.ext4 /dev/sda

mke2fs 1.44.5 (15-Dec-2018)
/dev/sda contains a ext4 file system
	last mounted on /mnt/ssd on Mon Sep  9 21:06:47 2019
Proceed anyway? (y,N) y
Creating filesystem with 58609664 4k blocks and 14655488 inodes
Filesystem UUID: 5c3a8481-682c-4834-9814-17dba166f591
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks):
done
Writing superblocks and filesystem accounting information: done     
```


**4. Manually mount the disk**

You can manually mount the disk to the directory `/mnt/ssd`.

```
pi@kube-master:~ $ sudo mkdir /mnt/ssd
pi@kube-master:~ $ sudo chown -R pi:pi /mnt/ssd/
pi@kube-master:~ $ sudo mount /dev/sda /mnt/ssd
```


**5. Automatically mount the disk on startup**

Next step consists to configure `fstab` to automatically mount the disk when the system starts.

You first need to find the Unique ID of the disk using the command `blkid`.

```
pi@kube-master:~ $ sudo blkid

/dev/mmcblk0p1: LABEL_FATBOOT="boot" LABEL="boot" UUID="F661-303B" TYPE="vfat" PARTUUID="a91dd8a2-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="8d008fde-f12a-47f7-8519-197ea707d3d4" TYPE="ext4" PARTUUID="a91dd8a2-02"
/dev/mmcblk0: PTUUID="a91dd8a2" PTTYPE="dos"
/dev/sda: UUID="0ac98c2c-8c32-476b-9009-ffca123a2654" TYPE="ext4"
```

Our SSD located in `/dev/sda` has a unique ID `0ac98c2c-8c32-476b-9009-ffca123a2654`.

Edit the file `/etc/fstab` and add the following line to configure auto-mount of the disk on startup.

```
pi@kube-master:~ $ sudo vi /etc/fstab
```

Add this line at the end:

```
UUID=0ac98c2c-8c32-476b-9009-ffca123a2654 /mnt/ssd ext4 defaults 0 0
```

Reboot the system

```
pi@kube-master:~ $ sudo reboot
```

You can verify the disk is correctly mounted on startup with the following command:

```
pi@kube-master:~ $ df -ha /dev/sda

Filesystem      Size  Used Avail Use% Mounted on
/dev/sda        458G   73M  435G   1% /mnt/ssd
```


**B. Share via NFS Server**

We now gonna make the directory `/mnt/ssd` of master accessible to other machines via NFS

**1. Install the required dependencies**

```
pi@kube-master:~ $ sudo apt-get install nfs-kernel-server -y
```


**2. Configure the NFS server**

Edit the file `/etc/exports` and add the following line

```
pi@kube-master:~ $ sudo vi /etc/exports

/mnt/ssd *(rw,no_root_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)
```


**3. Start the NFS Server**

```
pi@kube-master:~ $ sudo exportfs -ra
```



**====== Worker nodes only - Mount the NFS share ======**



**1. Install the necessary dependencies**

```
pi@kube-worker1:~ $ sudo apt-get install nfs-common -y
```

**2. Create the directory to mounty the NFS Share**

Create the directory `/mnt/ssd` and set the ownership to `pi`

```
pi@kube-worker1:~ $ sudo mkdir /mnt/ssd
pi@kube-worker1:~ $ sudo chown -R pi:pi /mnt/ssd/
```

**3. Configure auto-mount of the NFS Share**

In this step, we will edit `/etc/fstab` to tell the OS to automatically mount the NFS share into the directory `/mnt/ssd` when the machine starts.

```
pi@kube-worker1:~ $ sudo vi /etc/fstab
```

Add the following line where `192.168.0.22:/mnt/ssd` is the IP of `kube-master` and the NFS share path.

```
192.168.0.22:/mnt/ssd   /mnt/ssd   nfs    rw  0  0
```

**4. Reboot the system**

```
pi@kube-worker1:~ $ sudo reboot
```


### Conclusion

To conclude, we now have three secured, up-to-date and operational machines to build a Kubernetes cluster and easily self-host and maintain applications at home!

In the [next chapter](/2020/04/13/install-and-configure-a-kubernetes-cluster-w), we will see how to install Kubernetes with [Rancher **k3s**](https://k3s.io) on those three machines and deploy the necessary tools such as Package Manager (helm), Proxy and Load Balancer, Certificate Manager, etc. to safely and efficiently deploy our applications.
