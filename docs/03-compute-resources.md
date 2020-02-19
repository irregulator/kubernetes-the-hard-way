# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
</pre></td>
<td style="vertical-align:top"><pre>
exo privnet create kubernetes -s 10.240.0.1 -e 10.240.0.253 -m "255.255.255.0"
</pre></td></tr></table>


A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
</pre></td>
<td style="vertical-align:top"><pre>
???
</pre></td></tr></table>


> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall rule that allows internal communication across all protocols:


<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
</pre></td>
<td style="vertical-align:top"><pre>
exo firewall create kubernetes-allow-internal
exo firewall add kubernetes-allow-internal -p tcp -P 0-65535 -c 10.240.0.0/24
exo firewall add kubernetes-allow-internal -p udp -P 0-65535 -c 10.240.0.0/24
exo firewall add kubernetes-allow-internal -p icmp -P 0-65535 -c 10.240.0.0/24

exo firewall add kubernetes-allow-internal -p tcp -P 0-65535 -c 10.200.0.0/24
exo firewall add kubernetes-allow-internal -p udp -P 0-65535 -c 10.200.0.0/24
exo firewall add kubernetes-allow-internal -p icmp -P 0-65535 -c 10.200.0.0/24
</pre></td></tr></table>


Create a firewall rule that allows external SSH, ICMP, and HTTPS:


<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
</pre></td>
<td><pre>
exo firewall create kubernetes-allow-external
exo firewall add kubernetes-allow-external -p icmp -c 0.0.0.0/0
exo firewall add kubernetes-allow-external -p tcp -P 22,6443 -c 0.0.0.0/0
</pre></td></tr></table>


> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
</pre></td>
<td style="vertical-align:top"><pre>
exo firewall show kubernetes-allow-internal -O text
exo firewall show kubernetes-allow-external
</pre></td></tr></table>

> output

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

> exoscale output (redacted)

```
┼─────────┼────────────────────┼──────────────────┼──────────┼
│  TYPE   │       SOURCE       │       PORT       │ PROTOCOL │
┼─────────┼────────────────────┼──────────────────┼──────────┼
│ ingress │ CIDR 10.240.0.0/24 │ 0,0 (Echo Reply) │ icmp     │
│ ingress │ CIDR 10.200.0.0/24 │ 0,0 (Echo Reply) │ icmp     │
│ ingress │ CIDR 10.240.0.0/24 │ 0-65535          │ tcp      │
│ ingress │ CIDR 10.200.0.0/24 │ 0-65535          │ tcp      │
│ ingress │ CIDR 10.240.0.0/24 │ 0-65535          │ udp      │
│ ingress │ CIDR 10.200.0.0/24 │ 0-65535          │ udp      │
┼─────────┼────────────────────┼──────────────────┼──────────┼

┼─────────┼────────────────┼──────────────────┼──────────┼
│  TYPE   │     SOURCE     │       PORT       │ PROTOCOL │
┼─────────┼────────────────┼──────────────────┼──────────┼
│ ingress │ CIDR 0.0.0.0/0 │ 0,0 (Echo Reply) │ icmp     │
│ ingress │ CIDR 0.0.0.0/0 │ 22               │ tcp      │
│ ingress │ CIDR 0.0.0.0/0 │ 6443             │ tcp      │
┼─────────┼────────────────┼──────────────────┼──────────┼
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
</pre></td>
<td style="vertical-align:top"><pre>
exo eip create
</pre></td></tr></table>


Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
</pre></td>
<td style="vertical-align:top"><pre>
exo eip list -O text
</pre></td></tr></table>


> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

> exoscale output

```
c05b1471-d1ea-43f1-a67d-36b3dc24bcee  ch-gva-2  194.182.xxx.xxx   false []
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
</pre></td>
<td style="vertical-align:top">
<blockquote>
Ensure you have an ssh key named id_rsa_portal under ~/.ssh/
</blockquote>
<br>
<blockquote>
Kubernetes requires at least 2GB for master nodes and 1GB for worker nodes.
GCP n1-standard-1 have 4GB.
</blockquote>
<pre>
for i in 0 1 2; do
  exo vm create controller-${i} -t "Linux Ubuntu 18.04 LTS 64-bit" -p kubernetes -s kubernetes-allow-internal,kubernetes-allow-external -o small -k id_rsa_portal
done
</pre>

<blockquote>
Configure DHCP for the privnet
https://community.exoscale.com/documentation/compute/private-networks/
</blockquote>
<pre>
ssh ubuntu@194.182.xxx.xxx

;; no dhcp on eth1 which is the privnet iface
ip addr show eth1

sudo nano /etc/netplan/eth1.yaml

network:
  version: 2
  ethernets:
    eth1:
      dhcp4: true

sudo netplan apply
ip addr show eth1

3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:ea:14:00:8b:b3 brd ff:ff:ff:ff:ff:ff
    inet 10.240.0.138/24 brd 10.0.0.255 scope global dynamic eth1
</pre>
</td></tr></table>

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
</pre></td>
<td style="vertical-align:top">
<br>
<blockquote>
Ensure you have an ssh key named id_rsa_portal under ~/.ssh/
</blockquote>
<br>
<blockquote>
Kubernetes requires at least 2GB for master nodes and 1GB for worker nodes.
GCP n1-standard-1 have 4GB.
</blockquote>
<pre>
for i in 0 1 2; do
  exo vm create worker-${i} -t "Linux Ubuntu 18.04 LTS 64-bit" -p kubernetes -s kubernetes-allow-internal,kubernetes-allow-external -o small -k id_rsa_portal
done
</pre>

<blockquote>
Configure DHCP for the privnet
https://community.exoscale.com/documentation/compute/private-networks/
</blockquote>
<pre>
ssh ubuntu@194.182.xxx.xxx

;; no dhcp on eth1 which is the privnet iface
ip addr show eth1

sudo nano /etc/netplan/eth1.yaml

network:
  version: 2
  ethernets:
    eth1:
      dhcp4: true

sudo netplan apply
ip addr show eth1

3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:ea:14:00:8b:b3 brd ff:ff:ff:ff:ff:ff
    inet 10.240.0.138/24 brd 10.0.0.255 scope global dynamic eth1
</pre>
</td></tr></table>


### Verification

List the compute instances in your default compute zone:

<table style="width: 100%;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="width:50%;vertical-align:top"><pre>
gcloud compute instances list
</pre></td>
<td style="vertical-align:top"><pre>
exo vm list
</pre></td></tr></table>

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

> exoscale output

```
e17c726e-4548-4668-ac84-aaf95194b001    controller-1    Small   ch-gva-2        Running 194.xxx.xxx.xx
c17313c8-52c6-455e-b4db-2594ed6368e6    controller-0    Small   ch-gva-2        Running 194.xxx.xxx.xx
23e9e044-f77c-457a-8876-ac957155ad25    worker-2        Small   ch-gva-2        Running 194.xxx.xxx.xx
17888481-cd56-4845-a423-f6fbce7543be    worker-1        Small   ch-gva-2        Running 194.xxx.xxx.xx
de5922b6-2511-43f8-a9db-75c165b0542b    worker-0        Small   ch-gva-2        Running 194.xxx.xxx.xx
e5303af6-ade9-49c1-98d5-0b413babbc5a    controller-2    Small   ch-gva-2        Running 194.xxx.xxx.xx
```
> exoscale validate the ips

```
exo vm list -O json | jq ".[].ip_address" | tr -d '"' | xargs -n1 -P6 -I '{}' ssh ubuntu@'{}' -C 'echo $(hostname); ip addr show eth1 | grep global'
```
> exoscale output

```
worker-0
    inet 10.240.0.103/24 brd 10.240.0.255 scope global dynamic eth1
controller-0
    inet 10.240.0.241/24 brd 10.240.0.255 scope global dynamic eth1
controller-2
    inet 10.240.0.222/24 brd 10.240.0.255 scope global dynamic eth1
worker-1
    inet 10.240.0.31/24 brd 10.240.0.255 scope global dynamic eth1
controller-1
    inet 10.240.0.213/24 brd 10.240.0.255 scope global dynamic eth1
worker-2
    inet 10.240.0.139/24 brd 10.240.0.255 scope global dynamic eth1

```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as described in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1042-gcp x86_64)
...

Last login: Sun Sept 14 14:34:27 2019 from XX.XXX.XXX.XX
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
