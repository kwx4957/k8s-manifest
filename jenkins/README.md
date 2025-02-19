## Jenkins 설치 
```sh
helm repo add jenkinsci https://charts.jenkins.io
helm repo update

kubectl create ns jenkins

helm upgrade --install jenkins jenkinsci/jenkins --version 5.8.13 -n jenkins \
--set persistence.storageClass=nfs \
--set controller.serviceType=LoadBalancer \
--set controller.servicePort=80

secret=$(kubectl get secret -n jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}")
echo $(echo $secret | base64 --decode)

helm uninstall jenkins -n jenkins
```
https://www.jenkins.io/doc/book/installing/kubernetes/#install-jenkins-with-helm-v3  
https://github.com/jenkinsci/helm-charts  
