# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

<table style="width: 100vw;font-size: xx-small">
<tr><th>Google Cloud</th><th>Exoscale</th></tr>
<tr><td style="max-width:50vw;vertical-align:top"><pre>
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way &bsol;
    --region $(gcloud config get-value compute/region) &bsol;
    --format 'value(address)')

  kubectl config set-cluster kubernetes-the-hard-way &bsol;
    --certificate-authority=ca.pem &bsol;
    --embed-certs=true &bsol;
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin &bsol;
    --client-certificate=admin.pem &bsol;
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way &bsol;
    --cluster=kubernetes-the-hard-way &bsol;
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
</pre></td>
<td style="max-width:50vw;vertical-align:top"><pre>
export KUBERNETES_PUBLIC_ADDRESS=$(exo eip list -O json | jq ".[].ip_address" | tr -d '"')

  kubectl config set-cluster kubernetes-the-hard-way &bsol;
    --certificate-authority=ca.pem &bsol;
    --embed-certs=true &bsol;
    --server=https://$(echo $KUBERNETES_PUBLIC_ADDRESS):6443

  kubectl config set-credentials admin &bsol;
    --client-certificate=admin.pem &bsol;
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way &bsol;
    --cluster=kubernetes-the-hard-way &bsol;
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
</pre></td></tr></table>


## Verification

Check the health of the remote Kubernetes cluster:

```
kubectl get componentstatuses
```

> output

```
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes
```

> output

```
NAME       STATUS   ROLES    AGE    VERSION
worker-0   Ready    <none>   2m9s   v1.15.3
worker-1   Ready    <none>   2m9s   v1.15.3
worker-2   Ready    <none>   2m9s   v1.15.3
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
