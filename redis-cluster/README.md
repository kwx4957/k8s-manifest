## Redis 설치 
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis-cluster bitnami/redis --version 20.7.0

# 비밀번호 
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis-cluster -o jsonpath="{.data.redis-password}" | base64 -d)

# 클라이언트 파드 실행
kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:7.4.2-debian-12-r0 --command -- sleep infinity

kubectl exec -it redis-client -- bash

REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-cluster-master
REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-cluster-replicas
```
