To install a kubernetes cluster from scratch, we use kubeadm.

https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

TODO: automate this whole page :)

## Steps to do on both masters and slaves

```
# ip_forward needs to be enabled before docker starts otherwise docker
# sets the iptables foward poilcy to drop
cat <<EOF >/etc/sysctl.d/05-forwarding.conf 
net.ipv4.ip_forward=1
EOF
sysctl --load=/etc/sysctl.d/05-forwarding.conf 

curl -sSL  https://get.docker.io | sudo bash #yolo

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
sed -i 's/\(ExecStart=.*kubelet.*\)/\1 --cgroup-driver=cgroupfs/' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload
systemctl restart kubelet
```

## Then, for the master node

```
kubeadm init
```

If your LAN IP address is not on the same interface as the default route (eg, in VirtualBox where the route is out the private vbox NAT interface), use the `--apiserver-advertise-address=x.x.x.x` argument to the above command to hard code an address.

The command will output a bunch of garbage, and eventually right at the end give you a line starting with `kubeadm join`. This is the line you need for the workers. Save it but keep it secure. This token is effectively a root password to your cluster.

And then make the kubectl API easily accessible by making a .kube config directory:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now you can run `kubectl` commands from your terminal. You can also copy this config file to another PC to use kubectl remotely.

## For a worker

When you initialised the master, it gave you a `kubectl join` command. Run that on the worker.

Alternatively, you can use `kubeadm token create | tail -n 1` on the master to generate a token. Then, on your worker-to-be, run `kubeadm join --token <token> 10.0.0.123:6443` (where 10.0.0.123 is your master's IP address)

Now go follow the [networking instructions](network.md) to set up the network for the worker - the node won't be considered as ready until it has network.

Once network is set up, wait for a minute and run (back on the master)

```
kubectl get nodes
```
and you should see your worker(s) listed as Ready if it joined and the networking is set up properly.

