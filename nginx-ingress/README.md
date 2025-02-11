```sh
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v4.0.0/deploy/crds.yaml
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress --version 2.0.0 --set controller.kind=daemonset
```

https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/installation-with-helm/
