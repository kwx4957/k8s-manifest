## MetalLB 설치 
```sh
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

cat > ipPool.yaml <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
EOF

cat > L2Advertisement.yaml <<EOF
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: L2Advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - metallb-ip-pool
EOF

kubectl apply -f ipPool.yaml
kubectl apply -f L2Advertisement.yaml
```

https://metallb.io/installation/
