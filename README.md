# Kubernetes Core

Full configuration of NXTHDR's Kubernetes cluster.  
The configuration of services running on this cluster are located on [k8s-services](https://github.com/NXTHDR/k8s-services).

## Cluster and Network

1. Bootstrap the cluster

```sh
$ curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC=" \
   --node-ip=2a06:de00:50:ffff:ffff::191 \
   --advertise-address=2a06:de00:50:ffff:ffff::191 \
   --cluster-cidr=2a06:de00:50:1::/64 \
   --service-cidr=2a06:de00:50:1:ff00::/112 \
   --cluster-dns=2a06:de00:50:1:ff00::10 \
   --flannel-backend=none \
   --disable=coredns \
   --disable-network-policy \
   --disable=traefik \
   --disable=servicelb" sh -
```

2. Calico Operator and CRD

```sh
$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/tigera-operator.yaml
```

3. Calico + BGP Configuration

```sh
$ kubectl apply -f manifests/network/
```

4. Enable metrics reporting

```sh
$ kubectl patch felixconfiguration default --type merge --patch '{"spec":{"prometheusMetricsEnabled": true}}'
$ kubectl patch installation default --type=merge -p '{"spec": {"typhaMetricsPort":9093}}'
```

## DNS 

1. Etcd credentials secret 

```sh
$ kubectl create secret generic etcd-credentials \
      --from-literal=etcd-username='root' \
      --from-literal=etcd-password='PLEASE-CHANGE-ME!' \
      -n dns-system
```

2. Etcd - CoreDNS - ExternalDNS

```sh
$ kubectl apply -f manifests/dns/ns.yml
$ kubectl apply -f manifests/dns/
```

Note that the DNS upstream is [nat64.net](https://nat64.net/) which will give us an IPv6 for IPv4-only services.


## Load-balancer 

1. MetalLB CRD

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

2. MetalLB 

```sh
kubectl apply -f manifests/loadbalancer/
```

## Certificates

1. Cert-Manager CRD

```sh
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
```

2. Etcd credentials secret (same than `dns-system` namespace)

```sh
$ kubectl create secret generic etcd-credentials \
      --from-literal=etcd-username='root' \
      --from-literal=etcd-password='PLEASE-CHANGE-ME!' \
      -n cert-manager
```

3. Issuer

```sh
$ kubectl apply -f manifests/cert/
```

4. CoreDNS dns01 webhook

See our Cert-Manager dns01 [webhook](https://github.com/NXTHDR/cert-manager-webhook-coredns) for CoreDNS.

 ```sh
 $ helm upgrade --install \
        webhook-coredns \
        -n cert-manager \
        --set groupName='acme.nextheader.dev' \
        deploy/webhook-coredns/
```

## Monitoring

1. Namespace

```sh
$ kubectl apply -f manifests/monitoring/ns.yml
```

2. Grafana admin secret (change it!)

```sh 
kubectl create secret generic grafana -n monitoring-system \
    --from-literal=grafana-admin-password=admin \
    --from-literal=grafana-admin-user=admin
```

3. Monitoring stack

```sh
$ kubectl apply -f manifests/monitoring/
```

## Uninstall k3s 

```sh 
/usr/local/bin/k3s-uninstall.sh
```