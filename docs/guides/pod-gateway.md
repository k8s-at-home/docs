# Introduction

This is a guide on how to send all traffic from a group of pods
to a gateway pod. The gateway pod will then typically use a VPN
to route the traffic further.

## Requirements

- one or more namespaces where you deploy pods to be routed
- another namespace to deploy the gateway pod to. It is critical to deploy the
  gateway to a different namespace as the routed pods.
- (optional) VPN client settings (credentials, hostname, etc)

## Deploying the pod gateway

### Namespace

You need a namespace with the label `routed-gateway` set to true. In this
tutorial the namespace will be called `vpn`:

```yaml
# namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: vpn
  labels:
    routed-gateway: "true"
```

### pod-gateway Helm release

You need to deploy the
[pod-gateway](https://github.com/k8s-at-home/charts/tree/master/charts/stable/pod-gateway)
helm chart. Assuming you use GitOps to deploy the routed pods, you might
use the following template to deploy the gateway pod:

```yaml
# HelmRelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vpn-gateway
  labels:
spec:
  interval: 5m
  chart:
    spec:
      # renovate: registryUrl=https://k8s-at-home.com/charts/
      chart: pod-gateway
      version: 2.0.0
      sourceRef:
        kind: HelmRepository
        name: k8s-at-home-charts
        namespace: flux-system
      interval: 5m

  # See https://github.com/k8s-at-home/charts/blob/master/charts/pod-gateway/values.yaml
  values:
    routed_namespaces:
    - vpn
```

The above should deploy a gateway and a gateway-hook:

- the gateway-hook has the task of modifying created PODs in the `vpn`
  namespace to be configured to use the VPN gateway. You will find more
  details in
  [its git repository](https://github.com/k8s-at-home/gateway-admision-controller).
- the gateway provides a VXLAN tunnel, a DHCP server and a DNS server for
  client pods to connect to. Optionally it also runs the VPN client. You will
  find more details in
  [its git repository](https://github.com/k8s-at-home/pod-gateway).

### Test deployment

Then you can deploy a test deployment to test it:

```yaml
# TestDeployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: terminal
  labels:
    app: terminal
spec:
  replicas: 1
  selector:
    matchLabels:
      app: terminal
  template:
    metadata:
      labels:
        app: terminal
    spec:
      containers:
      - name: alpine
        image: alpine
        command:
        - /bin/sh
        - -c
        - while true; do
          sleep 600 &
          wait $!;
          done
```

If you exec into it you should see the traffic being routed throug the gateway:

```shell
root@k3s1:~# kubectl exec -ti -n vpn deploy/terminal -- sh
/ # ip route
default via 172.16.0.1 dev vxlan0
10.0.0.0/8 via 10.0.2.1 dev eth0
10.0.2.0/24 dev eth0 scope link  src 10.0.2.174
172.16.0.0/24 dev vxlan0 scope link  src 172.16.0.133
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if228: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether 6a:e8:0b:31:eb:3e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.174/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::68e8:bff:fe31:eb3e/64 scope link
       valid_lft forever preferred_lft forever
4: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc noqueue state UNKNOWN qlen 1000
    link/ether 06:81:5d:8c:4a:15 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.133/24 brd 172.16.0.255 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::481:5dff:fe8c:4a15/64 scope link
       valid_lft forever preferred_lft forever
/ # cat /etc/resolv.conf
nameserver 172.16.0.1

/ # ping -c1 example.com
64 bytes from 93.184.216.34: seq=0 ttl=115 time=36.366 ms

--- example.com ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 36.366/36.366/36.366 ms
```

The important part is that default gateway and the DNS are set to 172.16.0.1 which
is the default IP of the gateway POD in the vlxlan network. If this is the case
then you are ready for the (optional) VPN setup

If the ping does not work and you are using Calico please check the
section bellow.

### Network Policy

For additional precaution you should deploy a network policy into each routed
namespace to prevent traffic to leave the K8S cluster without passing through the
pod gateway. This way, even in case of setup errors, you will not leak any traffic.

```yaml
# NetworkPolicy.yaml
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: vpn-namespace
spec:
  podSelector: {}
  ingress:
  - from:
    # Only allow ingress from K8S
    - ipBlock:
        cidr: 10.0.0.0/8
  egress:
  - to:
    # Only allow egress to K8S
    - ipBlock:
        cidr: 10.0.0.0/8
  policyTypes:
    - Ingress
    - Egress
```

## Additional Configuration

### VPN

You will need to enable the vpn sidecar in the helm release settings:

```yaml
# HelmRelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vpn-gateway
spec:
  ...
  values:
    ...

    addons:
      vpn:
        enabled: true
        # You might use `openvpn` or `wireguard`
        type: openvpn
        openvpn:
        # VPN settings stored in secret `vpnConfig`. The secret mus have a key
        # a key called `vpnConfigfile` with the openvpn/wireguard config files in them
        configFileSecret: openvpn

        livenessProbe:
          exec:
            # In the example bellow the VPN output is in Belgic (BE) - change appropiatly
            command:
              - sh
              - -c
              - if [ $(wget -q -O- https://ipinfo.io/country) == 'BE' ]; then exit 0; else exit $?; fi
          initialDelaySeconds: 30
          periodSeconds: 60
          failureThreshold: 1

        networkPolicy:
          enabled: true

          egress:
            - to:
              - ipBlock:
                  cidr: 0.0.0.0/0
              ports:
              # VPN traffic port - change if your provider uses a different port
              - port: 443
                protocol: UDP
            - to:
                # Allow traffic within K8S - change if your K8S cluster uses a different CIDR
              - ipBlock:
                  cidr: 10.0.0.0/8
      settings:
        # tun0 for openvpn, wg0 for wireguard
        VPN_INTERFACE: tun0
        # Prevent non VPN traffic to leave the gateway
        VPN_BLOCK_OTHER_TRAFFIC: true
        # If VPN_BLOCK_OTHER_TRAFFIC is true, allow VPN traffic over this port
        VPN_TRAFFIC_PORT: 443
        # Traffic to these IPs will be send through the K8S gateway
        # change if your K8S cluster or home network uses a different CIDR
        VPN_LOCAL_CIDRS: "10.0.0.0/8 192.168.0.0/16"
```

### Exposing routed pod ports from the gateway

This is specially useful if you need to expose PODs to the Internet thorugh the
VPN server. For example, you might expose the torrent port or a web server.

You need to ensure the exposed routed pod has a static name. If you use the
k8s-at-home charts, this is done by setting the `hostname` variable:

```yaml
# HelmRelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: qbittorrent
spec:
  ... 
  values:
    hostname: torrent # needed for an static IP
```

Then you can expose the port by extending the pod-gateway helm-release:

```yaml
# HelmRelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vpn-gateway
spec:
  ...
  values:
    ...  
    # -- settings to expose ports, usually through a VPN provider.
    # NOTE: if you change it you will need to manually restart the gateway POD
    publicPorts:
    - hostname: torrent #hostname assigned to the pod
      IP: 10 # must be an integer between 2 and VXLAN_GATEWAY_FIRST_DYNAMIC_IP (20 by default)
      ports:
      - type: udp
        port: 18289
      - type: tcp
        port: 18289
```

### Calico

Calico only configures a single dwfault gateway and not dedicated rules for traffic
within the K8S cluster. So you will need to add that explictly by extending the
pod-gateway helm chart deployment:

```yaml
# HelmRelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: vpn-gateway
spec:
  ...
  values:
    ...
    settings:
    # Route internal K8s and local home traffic in to the defaullt K8S gateway
    - NOT_ROUTED_TO_GATEWAY_CIDRS: "172.22.0.0/12 <loca home network CIDR>"
    - VPN_LOCAL_CIDRS: "172.22.0.0/12 <local home network CIDR>"
    # Use a different VXLAN network segment that does not conflict with the above
    - VXLAN_IP_NETWORK: "192.168.242.0/24"
```

## How to debug

### Connectivity

From gateway and routed pods check the following pings:

1. K8S DNS IP (check an stardard port to find out)
   - if this fails, then likelly you forgot setting `NOT_ROUTED_TO_GATEWAY_CIDRS`.
     Please read the Calico setup.
2. from the routed pod, the gateway IP (172.16.0.1 by default)
   - if this fails, there is problem with the VXLAN setup and more debugging is needed
3. from the gateway pod, the routed pod
   - if this fails, there is problem with the VXLAN setup and more debugging is needed
4. an IP in your home network
   - if this fails, then likelly you forgot setting `VPN_LOCAL_CIDRS` to include
     your K8S CIDR - the default 10.0.0.0/8 works in Flannel (k3s default) but not
     in Calico.
5. a hostname in Internet
   - without VPN:
     - if this fails, you might have set the VPN_BLOCK_OTHER_TRAFFIC variable to
       true
   - with VPN:
     - if this fails, check the following:
       - output of the wireguard
       - VPN_INTERFACE - must be wg0 for wireguard
       - VPN_TRAFFIC_PORT - the default, 443 works for some openvpn servers
       - VPN_LOCAL_CIDRS - it must include the K8S and local home CIDRs
6. a hostname
   - if this fails, there is problem with the DNS resolution. More debugging is needed

### Collecting more debug

Collect the following from the routed and gateway pods:

- `ip route`
- `ip addr`
- `cat /etc/resolv.conf`
- `ping -c1 example.com`

You can compare with the output of the deployment example above. If you are not able
to tell what the problem is you might [open an issue](https://github.com/k8s-at-home/charts/issues/new/choose)
for chart `pod-gateway`. Please attach there the connectivity test results and the
additional debug information.
