# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Get internal IP address and Pod CIDR range for each worker instance and create services:

```bash
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)

  for HOST in server node-0 node-1 node-2; do
    cat <<EOF > ${HOST}-kubernetes-routes.service
[Unit]
Description=Kubernetes Pod Network Routes
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
EOF
    while read IP FQDN OTHER_HOST SUBNET; do
      if [ "$HOST" != "$OTHER_HOST" ]; then
        echo "ExecStart=/sbin/ip route replace ${SUBNET} via ${IP}" >> ${HOST}-kubernetes-routes.service
      fi
    done < <(grep node machines.txt)

    cat <<EOF >> ${HOST}-kubernetes-routes.service
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
  done
}
```

Copy and enable services:

```bash
{
  for HOST in server node-0 node-1 node-2; do
    scp ${HOST}-kubernetes-routes.service root@${HOST}:/etc/systemd/system/kubernetes-routes.service

    ssh root@${HOST} systemctl daemon-reload
    ssh root@${HOST} systemctl enable --now kubernetes-routes
  done
}
```

## Verification

```bash
ssh root@server ip route
```

```text
...
10.5.20.0/24 via XXX.XXX.XXX.XXX dev ens6
10.5.21.0/24 via XXX.XXX.XXX.XXX dev ens6
10.5.22.0/24 via XXX.XXX.XXX.XXX dev ens6
...
```

```bash
ssh root@node-0 ip route
```

```text
...
10.5.21.0/24 via XXX.XXX.XXX.XXX dev ens7
10.5.22.0/24 via XXX.XXX.XXX.XXX dev ens7
...
```

```bash
ssh root@node-1 ip route
```

```text
...
10.5.20.0/24 via XXX.XXX.XXX.XXX dev ens7
10.5.22.0/24 via XXX.XXX.XXX.XXX dev ens7
...
```

```bash
ssh root@node-2 ip route
```

```text
...
10.5.20.0/24 via XXX.XXX.XXX.XXX dev ens7
10.5.21.0/24 via XXX.XXX.XXX.XXX dev ens7
...
```


Next: [Smoke Test](12-smoke-test.md)
