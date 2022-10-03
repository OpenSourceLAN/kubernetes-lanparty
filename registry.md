Because you don't know which  host your container will run on, your images need to be able to be pulled from a docker registry. You can't just build an image on a host and trust that it will work.

To that end, instantiate a registry on your LAN.

Start a docker registry somewhere. This should even be in k8s itself, but that's left as an exercise for the reader at this time. Give it a good DNS name.

```
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

**Important**: this registry has no authentication or access restrictions. Consider your threat model before leaving this as is. [Follow instructions on this page](https://docs.docker.com/registry/deploying/) for how to secure the registry. And then [this other page](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) for how to configure kubernetes to support that registry.

With the registry up and running, now we need to configure the docker service on **every kubernetes host** to support our unsecured docker registry:

```
cat <<EOF > /etc/docker/daemon.json
{
  "insecure-registries": [ "docker.pax.lan"]
}
EOF
service docker restart
```
Make sure to update the hostname accordingly. `some.docker.registry` should be an FQDN that resolves to your registry host.

But that only impacts you if you're using `docker`... latest versions of Kube use `containerrd` interface, which needs to be configured seaprately:

```
cat <<EOF > /etc/containerd/config.toml
version = 2

[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.pax.lan"]
    endpoint = ["http://docker.pax.lan"]
EOF

```

Now make an image. On your local desktop or whatever, tag and push an image:
```
docker tag armagetron:latest some.docker.registry:5000/armagetron:latest
docker push some.docker.registry:5000/armagetron:latest
```

The image is now in the docker registry, and accessible by kubernetes.

```
kubectl run armagetron-test-server --image=some.docker.registry:5000/armagetron:latest -t -i --rm

```

## Securing the registry

```
docker run \
  --entrypoint htpasswd \
  registry:2 -Bbn testuser testpassword > auth/htpasswd
```

```
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v `pwd`/registry:/var/lib/registry \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

```
kubectl create secret docker-registry secretnamehere --docker-username=someuser --docker-password=somepassword --docker-server='https://some.docker.registry'
```

```
kubectl run -i -t hello-world --restart=Never --rm=true \
  --image=dockerreg.lanadmins.net/lan-registration:latest \
  --overrides='{ "apiVersion": "v1", "spec": { "imagePullSecrets": [{"name": "dockerreglanadminsnet"}] } }'
```

```
curl --basic --user someuser:somepassword https://some-secure-reg.com/v2/_catalog
curl --basic --user someuser:somepassword https://some-secure-reg.com/v2/someimage/tags/list
```

Make this secret default for all pods in a namespace using [this method](https://stackoverflow.com/a/40646132)
