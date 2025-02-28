## roki 설치
```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm show values grafana/loki --version 6.27.0 > values.yaml

vi values.yaml
---
global:
  dnsService: "coredns"

minio:
  enabled: false

loki:
  auth_enabled: false
  schemaConfig:
    configs:
      - from: "2099-99-99"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: index_
          period: 24h

  # chunk 압축 방식. 
  ingester:
    chunk_encoding: snappy

  # Default is 4, if you have enough memory and CPU you can increase, reduce if OOMing
  querier:
    max_concurrent: 4

  pattern_ingester:
    enabled: true

  limits_config:
    allow_structured_metadata: true
    volume_enabled: true
    retention_period: 1h # 데이터 보존 기관 

  # AWS s3구성
  storage:
    bucketNames:
      chunks: chunks
      ruler: ruler
      admin: admin

# Index files will be written locally at /loki/index and, eventually, will be shipped to the storage via tsdb-shipper.
  storage_config:
    aws:
      # Note: use a fully qualified domain name (fqdn), like localhost.
      # full example: http://loki:supersecret@localhost.:9000
      s3: http://{id}:{password}@minio.default.svc.cluster.local:9000/{bucket}
      s3forcepathstyle: true
    tsdb_shipper:
      active_index_directory: var/loki/index
      cache_location: var/loki/index_cache
      cache_ttl: 24h

test:
  enabled: false

lokiCanary:
  enabled: false

deploymentMode: SimpleScalable

backend:
  replicas: 3
read:
  replicas: 3
write:
  replicas: 3 # To ensure data durability with replication
---
helm install loki grafana/loki -f values.yaml --version 6.27.0
# 그라파나에 Add dataSource > loki > http://loki-gateway.monitoring.svc.cluster.local/
```

[공식 문서]  
https://grafana.com/docs/loki/latest/setup/install/helm/install-scalable/     
https://github.com/grafana/loki/blob/main/production/helm/loki/values.yaml     

[Blog]    
https://heekng.tistory.com/246    

## TroubleShooting
1. write,backend가 실행이 안됌
```sh
level=error ts=2025-02-26T06:17:39.009316425Z caller=log.go:216 msg="error running loki"
err="mkdir /loki/index: read-only file system\nerror initialising module: store\ngithub.com/grafana/dskit/modules.
(*Manager).initModule\n\t/src/loki/vendor/github.com/grafana/dskit/modules/modules.go:138\ngithub.com/grafana/dskit/modules.
(*Manager).InitModuleServices\n\t/src/loki/vendor/github.com/grafana/dskit/modules/modules.go:108\ngithub.com/grafana/loki/v3/pkg/loki.
(*Loki).Run\n\t/src/loki/pkg/loki/loki.go:495\nmain.main\n\t/src/loki/cmd/loki/main.go:129\nruntime

# Before
loki:
  tsdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/index_cache
# After
loki:
  tsdb_shipper:
    active_index_directory: var/loki/index
    cache_location: var/loki/index_cache
```
https://community.grafana.com/t/mkdir-loki-tsdb-shipper-active-read-only-file-system/127194/2

