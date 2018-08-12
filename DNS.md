## Using CoreDNS

CoreDNS is the DNS server in kubeadm 1.11 onwards. This supports [rewriting queries](https://coredns.io/plugins/rewrite/). We can [update the coredns config.](configs/coredns.yaml)

The linked config rewrites `blah.server.lan` to `blah.default.svc.cluster.local`. All that remains is exposing the DNS. We can run dnsmasq locally to forward external DNS queries to the internal address. 

And then [dns-reflector.yaml](configs/dns-reflector.yaml)  will run a DNS server on all kube master hosts which listens on the local LAN interface of the master (so it has a predictable address), and forwards queries _only_ for `.server.lan` to the kube dns system, and drops all other requests. 

### Making services appear

Assuming that you have some containers with the label `app: tron`, the below service will expose those containers to DNS in a conveniently short form.

```
apiVersion: v1
kind: Service
metadata:
  name: tron
spec:
  selector:
    app: tron
  clusterIP: None
  ports:
  - name: foo # These ports don't matter if you're not using SRV recrds, but I think you need one otherwise no A records are returned
    port: 1234
    targetPort: 1234
```

produces (with 2 replicas):

```
$ dig +short  @10.96.0.10 tron.default.svc.cluster.local
10.0.0.195
10.0.0.203
```

We can use the DNS rewriting config at the top of this file to then get `tron.server.lan` or similar as a nice human friendly name.


