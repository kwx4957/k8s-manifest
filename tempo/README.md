## Tempo 설치 
```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm show values grafana/tempo-distributed > values.yaml

vi valuess.yaml
---
global:
  dnsService: 'coredns'

storage:
  trace:
    backend: s3
    s3:
      access_key: 'grafana-tempo'
      secret_key: 'supersecret'
      bucket: 'tempo-traces'
      endpoint: 'tempo-minio:9000'
      insecure: true

# MinIO storage configuration
minio:
  enabled: true

  # distributed 으로 설정하면 sts가 pod를 16개를 생성한다. minio에 대한 설정을 잘 되어있으나 원인을 모르겠다. 
  # 기본적으로 sts가 16개를 생성하도록 되어있는 것으로 보인다. 
  mode: standalone
  rootUser: grafana-tempo
  rootPassword: supersecret
  buckets:
    # Default Tempo storage bucket
    - name: tempo-traces
      policy: none
      purge: false

# Specifies which trace protocols to accept by the gateway.
traces:
  otlp:
    grpc:
      enabled: true
    http:
      enabled: true
  zipkin:
    enabled: false
  jaeger:
    thriftHttp:
      enabled: false
  opencensus:
    enabled: false
---

helm install tempo grafana/tempo-distributed -f values.yaml --version 1.18.2

# 공식문서에는 admin에 대한 토큰을 포함하는 파드가 포함되어 있다고 했으나 파드를 발견하지 못하였음
kubectl get pods | awk '/.*-tokengen-job-.*/ {print $1}' | xargs -I {} kubectl logs {} | awk '/Token:\s+/ {print $2}'

# 그라파나에서 Connections > Add new connection > tmepo 검색 후 Tempo 추가
# http://tempo-query-frontend.trace-test.svc.cluster.local:3100

helm upgrade tempo grafana/tempo-distributed -f values.yaml 

helm uninstall tempo 
```
https://grafana.com/docs/helm-charts/tempo-distributed/next/get-started-helm-charts/   
https://github.com/grafana/helm-charts/tree/main/charts/tempo-distributed 
