# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

The commands in this lab must be run on each controller instance: `controller-0`, `controller-1`, and `controller-2`. Login to each controller instance using the `gcloud` command. Example:

<table style="width: 100vw;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="max-width:50vw;vertical-align:top"><pre>
gcloud compute ssh controller-0
</pre></td>
<td style="max-width:50vw;vertical-align:top"><pre>
exo ssh controller-0
</pre></td></tr></table>

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```
sudo mkdir -p /etc/kubernetes/config
```

### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl"
```

Install the Kubernetes binaries:

```
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### Configure the Kubernetes API Server

```
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
```

The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:

<table style="width: 100vw;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="max-width:50vw;vertical-align:top"><pre>
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
</pre></td>
<td style="max-width:50vw;vertical-align:top"><pre>
export INTERNAL_IP=$(ip addr show eth1  | grep -Po 'inet \K[\d.]+')
</pre></td></tr></table>

Create the `kube-apiserver.service` systemd unit file:


<table style="width: 100vw;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="max-width:50vw;vertical-align:top"><pre>
cat <&lt;EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver &bsol;
  --advertise-address=${INTERNAL_IP} &bsol;
  --allow-privileged=true &bsol;
  --apiserver-count=3 &bsol;
  --audit-log-maxage=30 &bsol;
  --audit-log-maxbackup=3 &bsol;
  --audit-log-maxsize=100 &bsol;
  --audit-log-path=/var/log/audit.log &bsol;
  --authorization-mode=Node,RBAC &bsol;
  --bind-address=0.0.0.0 &bsol;
  --client-ca-file=/var/lib/kubernetes/ca.pem &bsol;
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota &bsol;
  --etcd-cafile=/var/lib/kubernetes/ca.pem &bsol;
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem &bsol;
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem &bsol;
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 &bsol;
  --event-ttl=1h &bsol;
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml &bsol;
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem &bsol;
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem &bsol;
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem &bsol;
  --kubelet-https=true &bsol;
  --runtime-config=api/all &bsol;
  --service-account-key-file=/var/lib/kubernetes/service-account.pem &bsol;
  --service-cluster-ip-range=10.32.0.0/24 &bsol;
  --service-node-port-range=30000-32767 &bsol;
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem &bsol;
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem &bsol;
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
</pre></td>
<td style="max-width:50vw;vertical-align:top">

> Get the controllers internal ips, use that in `--etcd-servers`

<pre>
cat <&lt;EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver &bsol;
  --advertise-address=${INTERNAL_IP} &bsol;
  --allow-privileged=true &bsol;
  --apiserver-count=3 &bsol;
  --audit-log-maxage=30 &bsol;
  --audit-log-maxbackup=3 &bsol;
  --audit-log-maxsize=100 &bsol;
  --audit-log-path=/var/log/audit.log &bsol;
  --authorization-mode=Node,RBAC &bsol;
  --bind-address=0.0.0.0 &bsol;
  --client-ca-file=/var/lib/kubernetes/ca.pem &bsol;
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota &bsol;
  --etcd-cafile=/var/lib/kubernetes/ca.pem &bsol;
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem &bsol;
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem &bsol;
  --etcd-servers=https://10.240.0.241:2379,https://10.240.0.213:2379,https://10.240.0.222:2379 &bsol;
  --event-ttl=1h &bsol;
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml &bsol;
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem &bsol;
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem &bsol;
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem &bsol;
  --kubelet-https=true &bsol;
  --runtime-config=api/all &bsol;
  --service-account-key-file=/var/lib/kubernetes/service-account.pem &bsol;
  --service-cluster-ip-range=10.32.0.0/24 &bsol;
  --service-node-port-range=30000-32767 &bsol;
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem &bsol;
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem &bsol;
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
</pre></td></tr></table>


### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place:

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --leader-elect=true \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```


### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into place:

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the `kube-scheduler.yaml` configuration file:

```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Create the `kube-scheduler.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Controller Services

```
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

### Enable HTTP Health Checks


> Exoscale's Network Load Balancer is in the works. You can set up nginx anyway.

A [Google Network Load Balancer](https://cloud.google.com/compute/docs/load-balancing/network) will be used to distribute traffic across the three API servers and allow each API server to terminate TLS connections and validate client certificates. The network load balancer only supports HTTP health checks which means the HTTPS endpoint exposed by the API server cannot be used. As a workaround the nginx webserver can be used to proxy HTTP health checks. In this section nginx will be installed and configured to accept HTTP health checks on port `80` and proxy the connections to the API server on `https://127.0.0.1:6443/healthz`.



> The `/healthz` API server endpoint does not require authentication by default.

Install a basic web server to handle HTTP health checks:

```
sudo apt-get update
sudo apt-get install -y nginx
```

```
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

```
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
```

```
sudo systemctl restart nginx
```

```
sudo systemctl enable nginx
```

### Verification

```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

Test the nginx HTTP health check proxy:

```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

```
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Sat, 14 Sep 2019 18:34:11 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
X-Content-Type-Options: nosniff

ok
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

----

## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.

The commands in this section will effect the entire cluster **and only need to be run once from one of the controller nodes**.

```
gcloud compute ssh controller-0
```

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

The Kubernetes API Server authenticates to the Kubelet as the `kubernetes` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kubernetes` user:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

----


## The Kubernetes Frontend Load Balancer

> Exoscale doesn't yet have a network load balancer. We will use HAProxy as a tcp load balancer.

In this section you will provision an external load balancer to front the Kubernetes API Servers. The `kubernetes-the-hard-way` static IP address will be attached to the resulting load balancer.

> The compute instances created in this tutorial will not have permission to complete this section. **Run the following commands from the same machine used to create the compute instances**.


### Provision a Network Load Balancer

Create the external load balancer network resources:


```
export KUBERNETES_PUBLIC_ADDRESS=$(exo eip list -O json | jq ".[].ip_address" | tr -d '"')

exo firewall create kubernetes-lb-allow-all
;; TODO limit firewall rules
exo firewall add kubernetes-lb-allow-all -p tcp -P 0-65535 -c 0.0.0.0/0
exo firewall add kubernetes-lb-allow-all -p tcp -P 0-65535 -c 0.0.0.0/0 -e
exo vm create load-balancer -t "Linux Ubuntu 18.04 LTS 64-bit" -p kubernetes -s kubernetes-lb-allow-all -o micro -k id_rsa_portal 
exo eip associate $KUBERNETES_PUBLIC_ADDRESS load-balancer

;; configure haproxy
cat > haproxy.cfg <<EOF
global
  ssl-server-verify none

defaults
  mode http
  timeout connect 5000ms
  timeout client 50000ms
  timeout server 50000ms

backend controllers
  mode tcp
  balance roundrobin
  option ssl-hello-chk
  # use the controller's ips
  # TODO move this to privnet?
  server controller-0 194.182.163.52:6443 check
  server controller-1 194.182.162.6:6443 check
  server controller-2 194.182.163.135:6443 check

frontend lb
  bind *:6443
  option tcplog
  mode tcp
  default_backend controllers
  log /dev/log local0 debug
EOF

;; configure netplan

cat > 51-eip.yml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    lo:
      match:
        name: lo
      addresses:
        - 194.182.160.189/32
EOF

exo ssh load-balancer
sudo apt-get install -y haproxy

sudo ip addr add 194.182.160.189/32 dev lo

sudo mv 51-eip.yml /etc/netplan/51-eip.yml
sudo netplan apply

sudo mv haproxy.cfg /etc/haproxy/haproxy.cfg
sudo service haproxy restart
exit

;;test
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version

```



```
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  gcloud compute http-health-checks create kubernetes \
    --description "Kubernetes Health Check" \
    --host "kubernetes.default.svc.cluster.local" \
    --request-path "/healthz"

  gcloud compute firewall-rules create kubernetes-the-hard-way-allow-health-check \
    --network kubernetes-the-hard-way \
    --source-ranges 209.85.152.0/22,209.85.204.0/22,35.191.0.0/16 \
    --allow tcp

  gcloud compute target-pools create kubernetes-target-pool \
    --http-health-check kubernetes

  gcloud compute target-pools add-instances kubernetes-target-pool \
   --instances controller-0,controller-1,controller-2

  gcloud compute forwarding-rules create kubernetes-forwarding-rule \
    --address ${KUBERNETES_PUBLIC_ADDRESS} \
    --ports 6443 \
    --region $(gcloud config get-value compute/region) \
    --target-pool kubernetes-target-pool
}
```

### Verification

> The compute instances created in this tutorial will not have permission to complete this section. **Run the following commands from the same machine used to create the compute instances**.

Retrieve the `kubernetes-the-hard-way` static IP address:

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

Make a HTTP request for the Kubernetes version info:

```
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> output

```
{
  "major": "1",
  "minor": "15",
  "gitVersion": "v1.15.3",
  "gitCommit": "2d3c76f9091b6bec110a5e63777c332469e0cba2",
  "gitTreeState": "clean",
  "buildDate": "2019-08-19T11:05:50Z",
  "goVersion": "go1.12.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
