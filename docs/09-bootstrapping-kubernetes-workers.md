# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

The commands in this section must be run from the `jumpbox`.

Copy the Kubernetes binaries and systemd unit files to each worker instance:

```bash
for HOST in node-0 node-1 node-2; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conflist > 10-bridge.conflist

  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml

  scp 10-bridge.conflist kubelet-config.yaml \
  root@${HOST}:~/
done
```

```bash
for HOST in node-0 node-1 node-2; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@${HOST}:~/
done
```

```bash
for HOST in node-0 node-1 node-2; do
  scp \
    downloads/cni-plugins/* \
    root@${HOST}:~/cni-plugins/
done
```

The commands in the next section must be run on each worker instance: `node-0`, `node-1` and `node-2`. Login to the worker instance using the `ssh` command. Example:

```bash
ssh root@node-0
```

## Provisioning a Kubernetes Worker Node

### Install the OS dependencies

```bash
{
  apt-get update
  apt-get -y install socat conntrack ipset kmod
}
```

> The socat binary enables support for the `kubectl port-forward` command.

### Swap Configuration

In newer versions of Kubernetes and when using cgroup v2, swap is supported and can be left enabled on nodes. This allows the system to better handle memory pressure by swapping out rarely used memory pages.

The Kubelet in this lab is configured with `failSwapOn: false`, which allows it to start even if swap is enabled. Additionally, critical services like the Kubelet and the container runtime are protected from being swapped out by setting `MemorySwapMax=0` in their respective systemd unit files.

Verify if swap is enabled:

```bash
swapon --show
```

If swap is enabled, you can leave it as is. Kubernetes will manage memory and swap usage accordingly.

To ensure the system remains stable under memory pressure, configure the following kernel parameters:

```bash
{
  cat <<EOF > /etc/sysctl.d/kubernetes.conf
vm.swappiness = 60
vm.watermark_scale_factor = 2000
vm.min_free_kbytes = 500000
EOF
  sysctl --system -p
}
```

> For further information on tuning swap for Kubernetes, refer to the [Tuning Linux Swap for Kubernetes: A Deep Dive](https://kubernetes.io/blog/2025/08/19/tuning-linux-swap-for-kubernetes-a-deep-dive/) blog post.

### Create the installation directories

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

### Install the worker binaries

```bash
{
  mv -v crictl kube-proxy kubelet runc \
    /usr/local/bin/
  mv -v containerd containerd-shim-runc-v2 containerd-stress /bin/
  mv -v cni-plugins/* /opt/cni/bin/
}
```

### Configure CNI Networking

Create the `bridge` network configuration file:

```bash
mv -v 10-bridge.conflist 99-loopback.conf /etc/cni/net.d/
```

To ensure network traffic crossing the CNI `bridge` network is processed by `iptables`, load and configure the `br-netfilter` kernel module:

```bash
{
  modprobe br-netfilter
  echo "br-netfilter" >> /etc/modules-load.d/modules.conf
}
```

```bash
{
  cat <<EOF >> /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
  sysctl --system -p
}
```

### Configure containerd

Install the `containerd` configuration files:

```bash
{
  mkdir -p /etc/containerd/
  mv -v containerd-config.toml /etc/containerd/config.toml
  mv -v containerd.service /etc/systemd/system/
}
```

### Configure the Kubelet

Create the `kubelet-config.yaml` configuration file:

```bash
{
  mv -v kubelet-config.yaml /var/lib/kubelet/
  mv -v kubelet.service /etc/systemd/system/
}
```

### Configure the Kubernetes Proxy

```bash
{
  mv -v kube-proxy-config.yaml /var/lib/kube-proxy/
  mv -v kube-proxy.service /etc/systemd/system/
}
```

### Start the Worker Services

```bash
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```

Check if the kubelet service is running:

```bash
systemctl is-active kubelet
```

```text
active
```

Be sure to complete the steps in this section on each worker node, `node-0`, `node-1` and `node-2`, before moving on to the next section.

## Verification

Run the following commands from the `jumpbox` machine.

List the registered Kubernetes nodes:

```bash
ssh root@server \
  "kubectl get nodes \
  --kubeconfig admin.kubeconfig"
```

```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   2m     v1.34.3
node-1   Ready    <none>   1m     v1.34.3
node-2   Ready    <none>   10s    v1.34.3
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
