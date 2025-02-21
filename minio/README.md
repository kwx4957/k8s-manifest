## minio 설치
- minio(standalone)
- minio Operator(minio 설치를 위한 CRD)
- minio tenant(사용자 제공 환경)
  
```sh
# minio operator install with helm
helm repo add minio-operator https://operator.min.io
helm repo update
helm install operator minio-operator/operator \
  --namespace minio-operator --create-namespace

# values.yaml 다운로드
curl -sLo values.yaml https://raw.githubusercontent.com/minio/operator/master/helm/tenant/values.yaml

vi values.yaml
---
tenant:
  name: minio
  configSecret:
    accessKey: minio
    secretKey: minio

  pool:
    - servers: 1
      storageClassName: nfs
      volumesPerServer: 1
      
  metrics:
    enabled: true
---

# tenant 배포
helm install minio minio-operator/tenant -f values.yaml \
--namespace minio-tenant-dev --create-namespace 

# dev console에 대해 포트 포워딩 후 https으로 접근한다.

# helm upgrade
helm upgrade minio minio-operator/tenant -n minio-tenant-dev

# helm 릴리즈 삭제
helm uninstall minio -n minio-tenant-dev
```  
https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-operator-helm.html
