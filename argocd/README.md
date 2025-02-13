## ArgoCD 설치
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

export ARGOCD_PASSWORD=$(kubectl get secret argocd-initial-admin-secret  -n argocd -o jsonpath="{.data.password}" | base64 -d)

echo $ARGOCD_PASSWORD
```

https://argo-cd.readthedocs.io/en/stable/getting_started/#3-access-the-argo-cd-api-server
