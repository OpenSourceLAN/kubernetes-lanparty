# Helm!

## Installation

### Certificates
To set up a secure Helm installation, first create a Certificate Authority (CA) and
generate some certificates!

```
# Generate CA private key
openssl genrsa -out ./ca.key.pem 4096
# Generate public cert - fine to accept all default answers to questions
openssl req -key ca.key.pem -new -x509 -days 7300 -sha256 -out ca.cert.pem -extensions v3_ca
# Private key for server
openssl genrsa -out ./tiller.key.pem 4096
# Private key for client
openssl genrsa -out ./helm.key.pem 4096
# Cert signing request for server and client - defaults are okay again
openssl req -key tiller.key.pem -new -sha256 -out tiller.csr.pem
openssl req -key helm.key.pem -new -sha256 -out helm.csr.pem
# And then sign the requests
openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in tiller.csr.pem -out tiller.cert.pem -days 365
openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in helm.csr.pem -out helm.cert.pem  -days 365
```

Whew. All the certs are generated. The `*.key.pem` files are your secrets - don't
make those public!

Now install Tiller to your cluster.

### Tiller (server)

```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller --tiller-tls --tiller-tls-cert ./tiller.cert.pem --tiller-tls-key ./tiller.key.pem --tiller-tls-verify --tls-ca-cert ca.cert.pem
```

Verify that it works with the following command. It may take a minute for Tiller to
download its image and start, so be patient if it returns an error about not finding
a ready tiller pod.

```
helm ls --tls --tls-ca-cert ca.cert.pem --tls-cert helm.cert.pem --tls-key helm.key.pem
```

### Helm (client)

Copy your client certificates in to your `~/.helm` directory:

```
cp ca.cert.pem $(helm home)/ca.pem
cp helm.cert.pem $(helm home)/cert.pem
cp helm.key.pem $(helm home)/key.pem
```

Helm is meant to use those automatically since that's its default Helm home directory.
But my version doesn't. So set the env var and test it works:

```
export HELM_HOME=$(helm home)
helm ls --tls
```

Success. Now the client is set up!

## Run a server

Set up a Trackmania server with a provided name. Starting from this directory:

```
helm install --tls --name trackmania-demo --set server.name="Some Trackmania Server" ./trackmania-forever
```

This creates a deployment called `trackmania-demo` and passes a value to the 
`server.name` property.

To see the status of your deployments:

```
helm ls --tls
```

Done!