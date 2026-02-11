# Deploying Cluster Add-ons

In this lab you will deploy the [CoreDNS](https://coredns.io/) cluster add-on and the [Metrics Server](https://github.com/kubernetes-sigs/metrics-server).

## Prerequisites

The commands in this lab must be run from the `jumpbox`.

## Configure the API Aggregation Layer

To support the Metrics Server, you must enable the API Aggregation Layer on the Kubernetes API Server. We have prepared a systemd unit file `units/kube-apiserver-aggregation.service` that includes the necessary flags.

Copy the new unit file to the `server` machine:

```bash
scp units/kube-apiserver-aggregation.service root@server:~/
```

Login to the `server` machine:

```bash
ssh root@server
```

Replace the existing `kube-apiserver` configuration with the new one:

```bash
mv kube-apiserver-aggregation.service \
  /etc/systemd/system/kube-apiserver.service
```

Reload the systemd configuration and restart the API Server:

```bash
systemctl daemon-reload
systemctl restart kube-apiserver
```

Verify the API Server is running:

```bash
systemctl status kube-apiserver
```

Exit the `server` machine:

```bash
exit
```

## Install Helm

Install the Helm package manager to simplify the deployment of the Metrics Server.

Download and install Helm:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

## Deploy CoreDNS

Deploy the CoreDNS cluster add-on:

```bash
kubectl apply -f configs/coredns.yaml
```

## Deploy Metrics Server

To deploy the Metrics Server securely using your Cluster CA, you need to generate a dedicated certificate for it.

### Generate Metrics Server Certificate

Generate the certificate and private key using the `[metrics-server]` section already configured in `ca.conf`:

```bash
{
  openssl genrsa -out metrics-server.key 2048

  openssl req -new -key metrics-server.key -sha256 \
    -config ca.conf -section metrics-server \
    -out metrics-server.csr

  openssl x509 -req -days 3653 -in metrics-server.csr \
    -copy_extensions copyall \
    -sha256 -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out metrics-server.crt
}
```

Create a TLS Secret in the `kube-system` namespace:

```bash
kubectl create secret tls metrics-server-cert \
  --cert=metrics-server.crt \
  --key=metrics-server.key \
  --namespace kube-system
```

### Install Metrics Server with Helm

Add the Metrics Server Helm repository:

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
```

Install the Metrics Server using the custom values file `configs/metrics-server-values.yaml`. This configuration mounts the certificate you just generated and enables TLS verification using your CA.

```bash
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  -f configs/metrics-server-values.yaml \
  --set-file apiService.caBundle=ca.crt
```

## Verification

Verify the CoreDNS pods are running:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Verify that DNS resolution works by creating a temporary busybox pod:

```bash
kubectl run -it --rm --restart=Never busybox --image=busybox:1.37 -- nslookup kubernetes 2>/dev/null  | /bin/grep '^[SAN][a-z]*:'
```

```text             
Server:         10.32.0.10
Address:        10.32.0.10:53
Name:   kubernetes.default.svc.cluster.internal
Address: 10.32.0.1
```

Verify the Metrics Server pod is running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=metrics-server
```

Wait a few minutes for metrics to be collected, then check node resource usage:

```bash
kubectl top nodes
```

```text
NAME     CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)   
node-0   15m          0%       270Mi           7%          
node-1   15m          0%       243Mi           6%   
```

Next: [Cleaning Up](14-cleanup.md)
