For the purposes of game servers, we want our containers to have an IP address on the  local LAN. We do this via a bridge interface and CNI.

## Do network the easy way

Make sure *every* host in your cluster has a `br0` bridge interface that is bridged to the LAN.

Make sure *every* host in your cluster has `ip_forwarding` [enabled with a default allow forwarding iptables rule](https://github.com/OpenSourceLAN/dhcp-cni-plugin#how-to-use) -

```
cat <<EOF >/etc/sysctl.d/05-forwarding.conf 
net.ipv4.ip_forward=1
EOF
sysctl --load=/etc/sysctl.d/05-forwarding.conf 
iptables --policy FORWARD ACCEPT
```

Now apply the magic fongi file:

```
kubectl apply -f configs/dhcp-daemonset.yaml
```

Done :)

## Do network the manual way

When kubernetes runs a new pod, it first invokes the CNI plugins you've specified, which will add their own networking configuration to the network namespace of the pod. Don't think of this as kubernetes configuring anything - network isn't managed by kubernetes, and all configuration is entirely host local. Kubernetes invokes the CNI plugin, which then reaches back in to the container to configure it.

So on every host (kubelets and masters), we configure a single bridge and CNI network that uses the bridge like this:

```
cat <<EOF > /etc/netplan/10-bridge.conf
network:
  ethernets:
    eno1:
            dhcp4: false
  bridges:
          br0:
                  interfaces: [eno1]
                  # either dhcp4: true, or, a static address like below
                  addresses: [ "10.0.200.1/16"]
                  gateway4: "10.0.0.1"
                  nameservers:
                          addresses: [ 10.0.0.1 ]
  version: 2
EOF

netplan apply

mkdir -p /etc/cni/net.d
cat <<EOF> /etc/cni/net.d/10-bridge.conf 
{
  "name": "bridgenet",
  "type": "bridge",
  "bridge": "br0",
  "ipMasq": false,
  "isDefaultGateway": false,
  "hairpinMode": false,
  "ipam": {
    "type": "dhcp"
  }
}
EOF

```
(replacing eno1 with the host's local LAN interface).

A requirement of using bridges is that we set  `iptables --policy FORWARD ACCEPT`. This is because **all** packets leaving a kube container need to be evaluated by the iptables chains, so we need iptables to allow forward traffic. 
Some websites recommend disabling `bridge-nf-call-iptables`, but that will result in the packets not hitting iptables rules, which breaks serviceaddresses/endpoints. So we need iptables, and we need it to forward.

Since a recent Docker version, if Docker starts with `ip_forward` disabled, docker will enable it and also set the default FORWARD policy to DROP. So the solution is to set the ip_forward sysctl to 1 in `/etc/sysctl.d` _before_ Docker starts, so it leaves the FORWARD policy at its default (ACCEPT)

The DHCP IPAM system requries a daemon running to assign IPs. The CNI plugins repository provides us with this daemon, so we just need to compile it and run it.

```
sudo apt-get install golang
git clone https://github.com/containernetworking/plugins.git
cd plugins/plugins/ipam/dhcp
mkdir ~/go
export GOPATH=~/go
go get
go build
sudo ./dhcp daemon &
```

Alternatively, if DHCP seems wrong for you, you can do host local static ranges instead:

```
cat <<EOF > /etc/cni/net.d/10-bridge.conf
{
  "cniVersion": "0.2.0",
  "name": "bridgenet",
  "type": "bridge",
  "bridge": "br0",
  "ipMasq": false,
  "isDefaultGateway": false,
  "hairpinMode": false,
  "ipam": {
    "type": "host-local", "ranges": [[{ "subnet": "10.0.0.0/24", "rangeStart": "10.0.0.40", "rangeEnd": "10.0.0.54", "gateway": "10.0.0.1"}]],
    "routes": [{"dst": "0.0.0.0/0", "gw": "10.0.0.1"}],
    "dataDir": "/run/ipam-state"  
  }
}
EOF
```

Inside the ranges arrays... the first level is interface. Every top level array entry will generate an interface inside a container. Commonly this might be for IPv6 and IPv4 to coexist. The second level is ranges for a given interface. Multiple ranges can be specified at the second level, but I'm uncertain what the behaviour is (and I don't think it matters to me)


### On the `macvlan` plugin

Originally, I was using the macvlan CNI plugin. This is touted as supported by kubernetes in a number of places, and totally supported by docker. When using this, I could spin up containers and they'd be on the LAN as expected. But kubedns would not work, complaining about being unable to connect to 10.96.0.1, and then kube dashboard wouldn't work because kubedns as down.

10.96.0.1 is inaccessbile because macvlan interfaces bypass all iptables rules of the host. Kube relies heavily on iptables rules for numerous reasons, including routing to "Cluster IP addresses" such as 10.96.0.1. Using a bridge interface instead forces packets through the host iptables stack, allowing correct processing of packages. Yay!

