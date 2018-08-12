From https://docs.giantswarm.io/guides/install-kubernetes-dashboard/

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
$ kubectl create serviceaccount cluster-admin-dashboard-sa
$ kubectl create clusterrolebinding cluster-admin-dashboard-sa \
  --clusterrole=cluster-admin \
  --serviceaccount=default:cluster-admin-dashboard-sa
$ kubectl get secret | grep cluster-admin-dashboard-sa | awk '{print $1}' | xargs kubectl describe secret

```

Then on your local PC, with a correctly configured `.kube/config` file such that you can use other `kubectl` commands... 

```
$ kubectl proxy
```

Which will open a proxy listening on localhost:8001 .. and you can open the dashboard at ..

http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

Select token authentication, and provide the token from the cluster-admin-dashboard-sa serviceaccount you created earlier

